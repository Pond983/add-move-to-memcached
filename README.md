# Add show to memcached
このリポジトリでは、slab の中身を出力する命令を追加する。  
  
今回は「show」という命令を新たに追加し、それを受け取った場合に slab の中身について出力することにした。  

***
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

***
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

***
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
***
### LRU algorithm を応用する
slab class 内に item が存在しているが、slab->slots の先に順番に入っているわけではない様子。  
おそらく、

* key から item を入手する(get) -> hash から
* item の移動 -> LRU の list の中でポインタをつなぎなおす

という形になっているために、そもそも 「slabclass を使って item を管理する」 必要がないのだろう。  
そのため、特定の slab に所属する lru の中身を見ることで slab の中身を全て出力させてみることにした。 
  
item が来た際に、list に格納する関数である *do_item_link_q* 関数では以下のように item を格納している。
```
static void do_item_link_q(item *it)
{ /* item is the new head */
    item **head, **tail;
    assert((it->it_flags & ITEM_SLABBED) == 0);

    head = &heads[it->slabs_clsid];
    tail = &tails[it->slabs_clsid];
    assert(it != *head);
    assert((*head && *tail) || (*head == 0 && *tail == 0));
    it->prev = 0;
    it->next = *head;
    if (it->next)
        it->next->prev = it;
    *head = it;
    if (*tail == 0)
        *tail = it;
    sizes[it->slabs_clsid]++;
```
この関数では、list の先頭から item を格納している。  
また、if 分から察するに、list が空の時の tail の初期値は 0 になっている。  
  
  
次に、item がどの LRU に所属しているのかを確認する。  
item -> clsid の中身を適当なデータを格納させて、確認した。

とりあえず、process_get_commandを以下のように書き換えた。
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
                    fprintf(stderr, "item address: %p\n", (void *)it); ===> item のアドレスを出力させる
                    fprintf(stderr, "it->slabs_clsid: %d\n", ITEM_clsid(it)); ===> class id を出力させる
                    MEMCACHED_COMMAND_GET(c->sfd, ITEM_key(it), it->nkey,
                                          it->nbytes, ITEM_get_cas(it));
                    int nbytes = it->nbytes;
                    ;
                    nbytes = it->nbytes;
                    char *p = resp->wbuf;
                    memcpy(p, "VALUE ", 6);
                    p += 6;
                    memcpy(p, ITEM_key(it), it->nkey);
                    fprintf(stderr, "resp->wbuf: %s\n", resp->wbuf);
                    p += it->nkey;
                    p += make_ascii_get_suffix(p, it, return_cas, nbytes);
                    fprintf(stderr, "resp->wbuf: %s\n", resp->wbuf);
                    resp_add_iov(resp, resp->wbuf, p - resp->wbuf);
```
この状態で、適当なデータを格納して get command を使用した結果、次の通りになった.
```
item address: 0x7f5aa9ffaf70
it->slabs_clsid: 1
resp->wbuf: VALUE hoge3
resp->wbuf: VALUE hoge3 0 5
```
おそらく、clsid が 1 ということは、slabclass が 1 番目の中の、HOT_LRU (0) に所属しているということだと思われる。  

ということで、slabclass を確認していき、slabs の値が 1 のものだけ、LRU の中身を出力させるように改造することにした。  
ただし、LRU ようの heads & tails 配列は item.c 内でしか参照できないので、 item.c 内に、*lru_show* 関数を作成し、LRU の中身を出力させることにした。

作成した関数は次のとおりである。
```
// show each lru contents
void lru_show(int slab_id, int item_num)
{
    item *it;
    if (tails[slab_id + HOT_LRU] != 0)
    {
        fprintf(stderr, "HOT: \n");
        it = heads[slab_id + HOT_LRU];
        while (it != tails[slab_id + HOT_LRU]) 
        {
            fprintf(stderr, "\tkey: %s,\tvalue: %s\n", ITEM_key(it), ITEM_data(it));
            it = it->next;
        } 
        fprintf(stderr, "\tkey: %s,\tvalue: %s\n", ITEM_key(it), ITEM_data(it));
    }

    if (tails[slab_id + WARM_LRU] != 0)
    {
        fprintf(stderr, "WARM: \n");
        it = heads[slab_id + WARM_LRU];
        while (it != tails[slab_id + WARM_LRU]) 
        {
            fprintf(stderr, "\tkey: %s,\tvalue: %s\n", ITEM_key(it), ITEM_data(it));
            it = it->next;
        } 
        fprintf(stderr, "\tkey: %s,\tvalue: %s\n", ITEM_key(it), ITEM_data(it));
    }
    
    if (tails[slab_id + COLD_LRU] != 0)
    {
        fprintf(stderr, "COLD: \n");
        it = heads[slab_id + COLD_LRU];
        while (it != tails[slab_id + COLD_LRU]) 
        {
            fprintf(stderr, "\tkey: %s,\tvalue: %s\n", ITEM_key(it), ITEM_data(it));
            it = it->next;
        } 
        fprintf(stderr, "\tkey: %s,\tvalue: %s\n", ITEM_key(it), ITEM_data(it));
    }
}
```
本当は、冗長な関数ではなく、for 文とか使って簡潔に書いたほうがかっこいいのだろうが、分かりやすくするために、あえて冗長に書いている。  
本当に、あえてだ…  

このように作成した後に process_show_command 関数 (slabs.c)も次のように書き換えた。
```
/* show slab information */
/* This function called from proto_text.c */
void process_show_command(conn *c)
{
    int i = 0;

    while (++i < MAX_NUMBER_OF_SLAB_CLASSES - 1)
    {
        // output slabclass information like -vvv command
        fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\tslabs %u\n",
                i, slabclass[i].size, slabclass[i].perslab, slabclass[i].slabs);


        // if the slabclass has items, output item key and data.
        if (slabclass[i].slabs > 0){
            lru_show(i, slabclass[i].slabs);
        }
        
    }
}
```
このようにした後で、とりあえず実行してみた。
```
slab class   1: chunk size        96 perslab   10922    slabs 1
COLD:
        key: hoge,      value: fuga

slab class   2: chunk size       120 perslab    8738    slabs 0
slab class   3: chunk size       152 perslab    6898    slabs 0
slab class   4: chunk size       192 perslab    5461    slabs 0
slab class   5: chunk size       240 perslab    4369    slabs 0
slab class   6: chunk size       304 perslab    3449    slabs 0
slab class   7: chunk size       384 perslab    2730    slabs 0
slab class   8: chunk size       480 perslab    2184    slabs 0
slab class   9: chunk size       600 perslab    1747    slabs 1
COLD:
        key: abc,       value: abcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcd

slab class  10: chunk size       752 perslab    1394    slabs 0
slab class  11: chunk size       944 perslab    1110    slabs 0
slab class  12: chunk size      1184 perslab     885    slabs 1
COLD:
        key: 123,       value: abcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcdabcd

slab class  13: chunk size      1480 perslab     708    slabs 0
slab class  14: chunk size      1856 perslab     564    slabs 0
slab class  15: chunk size      2320 perslab     451    slabs 0
slab class  16: chunk size      2904 perslab     361    slabs 0
slab class  17: chunk size      3632 perslab     288    slabs 0
slab class  18: chunk size      4544 perslab     230    slabs 0
slab class  19: chunk size      5680 perslab     184    slabs 0
slab class  20: chunk size      7104 perslab     147    slabs 0
slab class  21: chunk size      8880 perslab     118    slabs 0
slab class  22: chunk size     11104 perslab      94    slabs 0
slab class  23: chunk size     13880 perslab      75    slabs 0
slab class  24: chunk size     17352 perslab      60    slabs 0
slab class  25: chunk size     21696 perslab      48    slabs 0
slab class  26: chunk size     27120 perslab      38    slabs 0
slab class  27: chunk size     33904 perslab      30    slabs 0
slab class  28: chunk size     42384 perslab      24    slabs 0
slab class  29: chunk size     52984 perslab      19    slabs 0
slab class  30: chunk size     66232 perslab      15    slabs 0
slab class  31: chunk size     82792 perslab      12    slabs 0
slab class  32: chunk size    103496 perslab      10    slabs 0
slab class  33: chunk size    129376 perslab       8    slabs 0
slab class  34: chunk size    161720 perslab       6    slabs 0
slab class  35: chunk size    202152 perslab       5    slabs 0
slab class  36: chunk size    252696 perslab       4    slabs 0
slab class  37: chunk size    315872 perslab       3    slabs 0
slab class  38: chunk size    394840 perslab       2    slabs 0
slab class  39: chunk size    524288 perslab       2    slabs 0
slab class  40: chunk size         0 perslab       0    slabs 0
slab class  41: chunk size         0 perslab       0    slabs 0
slab class  42: chunk size         0 perslab       0    slabs 0
slab class  43: chunk size         0 perslab       0    slabs 0
slab class  44: chunk size         0 perslab       0    slabs 0
slab class  45: chunk size         0 perslab       0    slabs 0
slab class  46: chunk size         0 perslab       0    slabs 0
slab class  47: chunk size         0 perslab       0    slabs 0
slab class  48: chunk size         0 perslab       0    slabs 0
slab class  49: chunk size         0 perslab       0    slabs 0
slab class  50: chunk size         0 perslab       0    slabs 0
slab class  51: chunk size         0 perslab       0    slabs 0
slab class  52: chunk size         0 perslab       0    slabs 0
slab class  53: chunk size         0 perslab       0    slabs 0
slab class  54: chunk size         0 perslab       0    slabs 0
slab class  55: chunk size         0 perslab       0    slabs 0
slab class  56: chunk size         0 perslab       0    slabs 0
slab class  57: chunk size         0 perslab       0    slabs 0
slab class  58: chunk size         0 perslab       0    slabs 0
slab class  59: chunk size         0 perslab       0    slabs 0
slab class  60: chunk size         0 perslab       0    slabs 0
slab class  61: chunk size         0 perslab       0    slabs 0
slab class  62: chunk size         0 perslab       0    slabs 0
```
適当にデータを突っ込んで、データを吐き出させてみた。  
結果、ITEM_key などもうまく機能し、key と value を出力させることに成功した。

***
### 考察1 get の時は HOT で、show の時は COLD ??
ただし、get で出力させたときに HOT 領域にいるのに、COLD 領域で出力されることがあった。
```
COLD:
        key: hoge,      value: fuga
...
it->slabs_clsid: 1 ===> COLD 領域にいるなら 129 になるはず (COLD_LRU(128) + slabclass_id(1) = 129)
resp->wbuf: VALUE hoge
resp->wbuf: VALUE hoge 0 4
```
おそらく、get コマンドで呼ばれた瞬間に HOT LRU に移動し、show コマンドを起動させるまでの間に COLD  領域に移動してしまっているためだと思われる。  
思われはするが、maintainer スレッドを見つけて、その動作中や動作後の挙動などを吐き出させるようにすれば、その実態が分かるかも。  
もしくは、関数自体が HOT 領域に移送しているのかも

***
### 考察1について
出力させている値をしっかり確認すると、get コマンド時に出力させる値に誤りがあった。  
今まで 「ITEM_clsid(it)」の値を出力させていたが、これは item がどの slab に所属しているかを示したものであり、  
正しくは 「it->slab_clsid」と書くべきであった。
(でもこれ分かりずらい気がする…。slab_clsid は lru の情報も内包したもので、ITEM_clsid は slab の情報だけって…。)  
  
とにもかくにも、これで正しい clsid 情報が分かる。

```
it->ITEM_clsid(it): 1
it->slabs_clsid: 129
resp->wbuf: VALUE hoge2
resp->wbuf: VALUE hoge2 0 5

slab class   1: chunk size        96 perslab   10922    slabs 1
COLD:
        key: hoge2,     value: fuga2
```

とりあえず、どちらも COLD 領域にいることを示してくれたので、これで整合性はとれた。

ただ…、あれ??  
最初は HOT 領域に入るんじゃないの??

ということで、set 時に HOT 領域に入っているか確認。  
*do_item_alloc* 関数で新規アイテムを LRU に突っ込んでいるとのことなので、
その部分を書き換えて、HOT 領域に入っているかを確認して見る。
```
/* Items are initially loaded into the HOT_LRU. This is '0' but I want at
     * least a note here. Compiler (hopefully?) optimizes this out.
     */
    if (settings.temp_lru &&
        exptime - current_time <= settings.temporary_ttl)
    {
        id |= TEMP_LRU;
    }
    else if (settings.lru_segmented)
    {
        id |= HOT_LRU;
    }
    else
    {
        /* There is only COLD in compat-mode */
        id |= COLD_LRU;
    }
    it->slabs_clsid = id;
    fprintf(stderr, "(do_item_alloc)it->slabs_clsid: %d\n", it->slabs_clsid);
```
こんな感じで、最後の最後に clsid がどこに設定されているかを出力するようにした。
この状態で、set コマンドを使用すると
```
(do_item_alloc)it->slabs_clsid: 1
item address: 0x7f374d0d0f70
it->ITEM_clsid(it): 1
it->slabs_clsid: 129
resp->wbuf: VALUE hoge
resp->wbuf: VALUE hoge 0 4
```
どうやら、set コマンドを使用したときには HOT 領域に入っているようだが、
その後 get コマンドを行うまでの間に COLD 領域に移送させられてるっぽい…。  

つまり、
* set された item は HOT 領域に所属する
* ただし、一瞬で maintainer thread (?) などによって、COLD 領域に突っ込まれる
* 結果、すぐに get や show コマンドを使用しても COLD 領域に入っている item しか確認できない。

ということだと、思われる。

これを調べるなら…、maintainer thread を止めるか、移送の際に出力をさせるようにすればよさそうだ。  
ということで、次は maintainer thread の動きについて調べてみようと思う。

***
### cold 領域への移動部分
lru_maintainer thread で移送を行っていることは分かっている。  
プログラムを追っていくと、item.c の *lru_pull_tail* 関数で移送が行われていそうなことが分かった。  
該当部は以下のようになっている。
```
if (move_to_lru)
        {
            fprintf(stderr, "In lru_pull_tail before:\tkey: %s,\tvalue: %s,\tit->slabs_clsid: %d\n", ITEM_key(it), ITEM_data(it), it->slabs_clsid);
            it->slabs_clsid = ITEM_clsid(it);
            it->slabs_clsid |= move_to_lru;
            item_link_q(it);
            fprintf(stderr, "In lru_pull_tail after:\tkey: %s,\tvalue: %s,\tit->slabs_clsid: %d\n", ITEM_key(it), ITEM_data(it), it->slabs_clsid);
        }
```
*move_to_url* に移送先の url が指定されており、指定された item の link をつなぎなおすことで実装を行っている様子。  
そのため、今までと同じように出力させてあげると次のようになった。
```
(do_item_alloc)it->slabs_clsid: 1
In lru_pull_tail before:        key: hoge,      value: fuga
,       it->slabs_clsid: 1
In lru_pull_tail after: key: hoge,      value: fuga
,       it->slabs_clsid: 129
(do_item_alloc)it->slabs_clsid: 1
In lru_pull_tail before:        key: hoge2,     value: fuga2
,       it->slabs_clsid: 1
In lru_pull_tail after: key: hoge2,     value: fuga2
,       it->slabs_clsid: 129
(do_item_alloc)it->slabs_clsid: 1
In lru_pull_tail before:        key: hoge3,     value: fuga3
,       it->slabs_clsid: 1
In lru_pull_tail after: key: hoge3,     value: fuga3
,       it->slabs_clsid: 129
```
なぜだか変な部分で改行されているが、set 後にすぐ COLD 領域へ移送されていることが分かる。  
このため、set されてすぐ lru_maintainer で移送されていることが確認することが出来た。