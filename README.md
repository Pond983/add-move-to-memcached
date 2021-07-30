# Add show to memcached
このリポジトリでは、slab の中身を出力する命令を追加する。  
  
今回は「show」という命令を新たに追加し、それを受け取った場合に slab の中身について出力することにした。  
  
## show 命令の追加
まず、show 命令を認識させることから始める。  
memcached では、命令は proto_test.c の process_command という関数で識別を行っている。

その中で 最初の文字が s となる命令群の中に、show という命令を認識できるよう書き換えた。

```
else if (first == 's')
    {
        if (strcmp(tokens[COMMAND_TOKEN].value, "set") == 0 && (comm = NREAD_SET))
        {

            WANT_TOKENS_OR(ntokens, 6, 7);
            process_update_command(c, tokens, ntokens, comm, false);
        }
        else if (strcmp(tokens[COMMAND_TOKEN].value, "stats") == 0)
        {

            process_stat(c, tokens, ntokens);
        }
        else if (strcmp(tokens[COMMAND_TOKEN].value, "shutdown") == 0)
        {

            process_shutdown_command(c, tokens, ntokens);
        }
        else if (strcmp(tokens[COMMAND_TOKEN].value, "slabs") == 0)
        {

            process_slabs_command(c, tokens, ntokens);
        }
        else if (strcmp(tokens[COMMAND_TOKEN].value, "show") == 0)
        {

            out_string(c, "show received");
            // int i=0;
            // while (++i < MAX_NUMBER_OF_SLAB_CLASSES - 1)
            // {
            //     fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
            //             i, slabclass[i].size, slabclass[i].perslab);
            // }
        }
        else
        {
            out_string(c, "ERROR");
        }
	}
```

とりあえず、受け取ったことを client 側に伝える形にした。  

その後、実行を行った結果はこんな感じ    

client side  
```
show
show received
```
  
まずは show を認識させられたっぽい。

## slab class の情報について出力
次に slab class の情報を出力させてみる。  
幸い -vvv  コマンドで起動させると以下のような情報が出力される。  

```
slab class   1: chunk size        96 perslab   10922
slab class   2: chunk size       120 perslab    8738
slab class   3: chunk size       152 perslab    6898
・・・
```
  
ということで、この情報を同じように出力させてみる。  
  
memcached では slab についての処理を slab.c というファイルで行っている。  
slabclass_t という構造体で、それぞれの slab を管理しており、slabclass という配列で
それぞれの slabclass_t 構造体を管理している。
この slabclass は slab.c 内でのみ参照可能となっている。  
  
そのため、今回は slab.c に process_show_command という関数を実装し、  
その中で -vvv コマンドのように出力を行わせることにした。  

ちなみに、slab.h の関数は memcached.h が include されていれば見れるっぽい。  

まずは、slab.c に以下のように関数を作成  
```
/* show slab information */
/* This function called from proto_text.c */
void process_show_command(conn *c){
    int i=0;
    while (++i < MAX_NUMBER_OF_SLAB_CLASSES - 1)
    {
        fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
                i, slabclass[i].size, slabclass[i].perslab);
    }
}
```

その後、slab.h に関数のプロトタイプ宣言を追加
```
/* show command function that output information about slabs */
void process_show_command(conn *c);
```

最後に prot_text.c の show 関数部を変更
```
else if (strcmp(tokens[COMMAND_TOKEN].value, "show") == 0)
        {

            out_string(c, "show received");
            process_show_command(c);
        }
```

さてさてどうなるかと、実行した  
  
surver side
```
<28 show
>28 show received
28: going from conn_parse_cmd to conn_new_cmd
slab class   1: chunk size        96 perslab   10922
slab class   2: chunk size       120 perslab    8738
slab class   3: chunk size       152 perslab    6898
slab class   4: chunk size       192 perslab    5461
slab class   5: chunk size       240 perslab    4369
slab class   6: chunk size       304 perslab    3449
slab class   7: chunk size       384 perslab    2730
slab class   8: chunk size       480 perslab    2184
slab class   9: chunk size       600 perslab    1747
slab class  10: chunk size       752 perslab    1394
slab class  11: chunk size       944 perslab    1110
slab class  12: chunk size      1184 perslab     885
slab class  13: chunk size      1480 perslab     708
slab class  14: chunk size      1856 perslab     564
slab class  15: chunk size      2320 perslab     451
slab class  16: chunk size      2904 perslab     361
slab class  17: chunk size      3632 perslab     288
slab class  18: chunk size      4544 perslab     230
slab class  19: chunk size      5680 perslab     184
slab class  20: chunk size      7104 perslab     147
slab class  21: chunk size      8880 perslab     118
slab class  22: chunk size     11104 perslab      94
slab class  23: chunk size     13880 perslab      75
slab class  24: chunk size     17352 perslab      60
slab class  25: chunk size     21696 perslab      48
slab class  26: chunk size     27120 perslab      38
slab class  27: chunk size     33904 perslab      30
slab class  28: chunk size     42384 perslab      24
slab class  29: chunk size     52984 perslab      19
slab class  30: chunk size     66232 perslab      15
slab class  31: chunk size     82792 perslab      12
slab class  32: chunk size    103496 perslab      10
slab class  33: chunk size    129376 perslab       8
slab class  34: chunk size    161720 perslab       6
slab class  35: chunk size    202152 perslab       5
slab class  36: chunk size    252696 perslab       4
slab class  37: chunk size    315872 perslab       3
slab class  38: chunk size    394840 perslab       2
slab class  39: chunk size    524288 perslab       2
slab class  40: chunk size         0 perslab       0
slab class  41: chunk size         0 perslab       0
slab class  42: chunk size         0 perslab       0
slab class  43: chunk size         0 perslab       0
slab class  44: chunk size         0 perslab       0
slab class  45: chunk size         0 perslab       0
slab class  46: chunk size         0 perslab       0
slab class  47: chunk size         0 perslab       0
slab class  48: chunk size         0 perslab       0
slab class  49: chunk size         0 perslab       0
slab class  50: chunk size         0 perslab       0
slab class  51: chunk size         0 perslab       0
slab class  52: chunk size         0 perslab       0
slab class  53: chunk size         0 perslab       0
slab class  54: chunk size         0 perslab       0
slab class  55: chunk size         0 perslab       0
slab class  56: chunk size         0 perslab       0
slab class  57: chunk size         0 perslab       0
slab class  58: chunk size         0 perslab       0
slab class  59: chunk size         0 perslab       0
slab class  60: chunk size         0 perslab       0
slab class  61: chunk size         0 perslab       0
slab class  62: chunk size         0 perslab       0
```
よくよく考えたら -vvv コマンドと同じようにやったら出力される結果は server side だった。  
出来れば client side に出力させたいが、まあ良しとしよう。  

## slab 内のデータを出力
さて、ここまで来たらそれぞれの slab の中身を slab の個数分出力させればいけそうだ。  

ということで、process_get_command と同じように item のデータの出力を行わせてみるとしよう  
  
process_show_command の中身を以下のように書き換えてみた
```
void process_show_command(conn *c)
{
    int i = 0;
    slabclass_t *p;
    item *it;
    mc_resp *resp = c->resp;

    while (++i < MAX_NUMBER_OF_SLAB_CLASSES - 1)
    {
        fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
                i, slabclass[i].size, slabclass[i].perslab);

        p = &slabclass[i];
        it = (item *)p->slots;
        for (int j = 0; j < p->slabs; j++)
        {
            // fprintf(stderr, "item%d, key: %s, value: %s\n", j, ITEM_key(it), ITEM_data(it));
            MEMCACHED_COMMAND_GET(c->sfd, ITEM_key(it), it->nkey,
                                  it->nbytes, ITEM_get_cas(it));
            // int nbytes = it->nbytes;
            ;
            // nbytes = it->nbytes;
            char *p = resp->wbuf;
            memcpy(p, "VALUE ", 6);
            p += 6;
            memcpy(p, ITEM_key(it), it->nkey);
            p += it->nkey;
            fprintf(stderr, "resp->wbuf: %s\n", resp->wbuf);
            // p += make_ascii_get_suffix(p, it, false, nbytes);
            resp_add_iov(resp, resp->wbuf, p - resp->wbuf);
        }
    }
}
```

若干分かりずらくなってしまったが、おそらく ITEM_key が item の key 情報を入手する macro であると予想  
試しに 4byte のデータを入れて出力してみた。  
実行した結果
```
slab class   1: chunk size        96 perslab   10922
resp->wbuf: VALUE
```
おろ??  
使い方が違うのかもしれない。  
試しに get コマンドのほうでも出力を行わせた。  
  
process_get_command を以下のように書き換えた
```
if (it)
            {
                /*
                 * Construct the response. Each hit adds three elements to the
                 * outgoing data list:
                 *   "VALUE "
                 *   key
                 *   " " + flags + " " + data length + "\r\n" + data (with \r\n)
                 */

                {
                    MEMCACHED_COMMAND_GET(c->sfd, ITEM_key(it), it->nkey,
                                          it->nbytes, ITEM_get_cas(it));
                    int nbytes = it->nbytes;
                    ;
                    nbytes = it->nbytes;
                    char *p = resp->wbuf;
                    memcpy(p, "VALUE ", 6);
                    p += 6;
                    memcpy(p, ITEM_key(it), it->nkey);
                    fprintf(stderr, "resp->wbuf: %s\n", resp->wbuf); ===> stderr に buffer の中身を出力させてみる
                    p += it->nkey;
                    p += make_ascii_get_suffix(p, it, return_cas, nbytes);
                    fprintf(stderr, "resp->wbuf: %s\n", resp->wbuf); ===> stderr に buffer の中身を出力させてみる
                    resp_add_iov(resp, resp->wbuf, p - resp->wbuf);
```

さて、実行結果は…
```
resp->wbuf: VALUE hoge
resp->wbuf: VALUE hoge 0 4
```

なぬ????