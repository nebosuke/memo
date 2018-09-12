## Apache httpd 2.2 の server-status についてのメモ

### 各状態について
よく観測されるものを整理。
- _ リクエスト待ち。プロセスは起動しており prefork の場合、accept(2)でソケットに接続されることを待っている状態。
- R リクエスト読み込み中。accept(2)した後、HTTPリクエストの読み込みが終わるまで。
- W リクエスト処理中。リクエストされたパスから適切なハンドラを選択しハンドラ内での処理、またソケットに対してのデータの書き込みを行っている状態。
- L ログ出力中。送信が終わったタイミングで所要時間などをログに記録する。
- K Keep-Alive中。リクエストの処理は終わったもののKeep-Aliveでソケットを開いたままにしていて次のリクエストを待っている状態。

RからWに変わるタイミングは以下の部分。ソケットの接続が完了した後に prefork の場合は、子プロセスにソケットを渡して子プロセス内で ap_process_http_connection が呼び出される。

```modules/http/http_core.c (Apache 2.2.34)```
```C
static int ap_process_http_connection(conn_rec *c)
{
    request_rec *r;
    apr_socket_t *csd = NULL;

    /*
     * Read and process each request found on our connection
     * until no requests are left or we decide to close.
     */
    
    // ここでスコアボードの状態を R にする
    ap_update_child_status(c->sbh, SERVER_BUSY_READ, NULL);
    
    while ((r = ap_read_request(c)) != NULL) { // ソケットからHTTPのリクエストを読み込む

        c->keepalive = AP_CONN_UNKNOWN;
        /* process the request if it was read without error */

        // ここでスコアボードの状態を W にする
        ap_update_child_status(c->sbh, SERVER_BUSY_WRITE, r);
        if (r->status == HTTP_OK)
            ap_process_request(r); // この先がアプリケーションの処理とクライアントへのデータの送信となる

        if (ap_extended_status)
            ap_increment_counts(c->sbh, r);

        if (c->keepalive != AP_CONN_KEEPALIVE || c->aborted)
            break; // KeepAliveを設定していないのでここで抜ける

        ap_update_child_status(c->sbh, SERVER_BUSY_KEEPALIVE, r);
        apr_pool_destroy(r->pool);

        if (ap_graceful_stop_signalled())
            break;

        if (!csd) {
            csd = ap_get_module_config(c->conn_config, &core_module);
        }
        apr_socket_opt_set(csd, APR_INCOMPLETE_READ, 1);
        apr_socket_timeout_set(csd, c->base_server->keep_alive_timeout);
        /* Go straight to select() to wait for the next request */
    }

    return OK;
}
```

### R状態のプロセスが大量にあるとき
- TCP 3way handshake が成功してソケットは ESTABLISHED になったが、クライアントからHTTPのリクエストデータがこない
- スロットをR状態で埋めてしまった場合はそれ以上のリクエストを処理できず、DoSに至る。
- accept(2) まではできているので、SYN flood攻撃のような状態とは異なる。
- Slowloris攻撃と同じような状態 reqtimeout_module が使えるなら影響を緩和できるかも。

### TCP_DEFER_ACCEPT の影響
- HTTPはクライアント側から接続しリクエストも最初はクライアントから送信する。
- 通常のTCPのシーケンスだと、3way handshake が成功したタイミングで、accept() が完了し、その後にHTTPリクエストがペイロードとして含まれるデータパケットを read(2) で読み込もうとする。
- accept(2) から read(2) までの間が R 状態
- プロセスをこの状態に留め置くことは、リソースの無駄遣いなので、Linux環境下でのApache httpdは、ソケットに TCP_DEFER_ACCEPT フラグをセットする。
- このフラグは Linux 特有のもので、クライアント側から最初のデータが送信されるプロトコルの場合、データパケットを受信するまで accept(2) を遅らせるというもの。
- データパケットの到着をカーネル内でバッファリングすることで、httpdプロセスの待ち時間を減らしリソースを最大限活用しようというコンセプトで設計されている。
- **TCP_DEFER_ACCEPTが設定されたソケットでは、accept(2) が成功した時には、すでにバッファにデータが存在するので、直後の read(2) が即座に成功するはず。つまり R になる瞬間は一瞬で server-status で見えることはほぼ無いはず。**

### TCP_DEFER_ACCEPT が設定されているのに R 状態となる
- accept(2) のまま長時間経過したとき (2.6.32以降 or バックポートされたカーネル)
- tcp backlog がデータパケット待ちの接続で埋まりこれ以上カーネル内でキープできないとき