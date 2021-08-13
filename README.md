# Add move to memcached
このリポジトリでは、slab の中身を出力する命令を追加する  
今回は「move」という命令を新たに追加し、それを受け取った場合に指定された item を指定された slabclass に移送させることにした
***
## 基本方針
* 「move {keyの名称} {移送先の slabclass の id}」という形で client 側からの命令を受け取る
* *item_get* 関数で指定の item のアドレスを入手する
* *do_item_alloc_pull* 関数で item の領域を確保する
* 元の item のデータをコピーしてくる
* 元の item を削除する
* 新規の item を LRU と hash にぶち込む!

***
## move 命令の追加
まずは add 命令の追加と同じように move 命令を認識するようにした。  
proto_text.c の *process_command* 関数に以下のような文を追加した。
```
else if (strcmp(tokens[COMMAND_TOKEN].value, "move") == 0)
    {
        out_string(c, "move received");
    }
```
この状態で move 命令を実行すると、client 側は以下のようになった。
```
move
move received
```
***
## *item_get* 関数を使って、item のアドレスを入手する
新たに *process_move_command* を用意した
```
static void process_move_command(conn *c, token_t *tokens, const size_t ntokens)
{
    char *key;
    size_t nkey;
    item *pre_it;
    key = tokens[KEY_TOKEN].value;
    nkey = tokens[KEY_TOKEN].length;

    pre_it = item_get(key, nkey, c, false);

    fprintf(stderr, "In move command: key: %s,\tvalue: %s\n", ITEM_key(pre_it), ITEM_data(pre_it));
    return;
}
```
この状態で move 命令を実行した結果、以下のようになった。
```
In move command: key: hoge,     value: fuga
```
***
## *do_item_alloc_pull* 関数で item の領域を確保する
関数内で、指定した slabclass から item を取得してくる  

*item_alloc* 関数をたどった結果、*do_item_alloc_pull* 関数で  
新規の item のアドレスを取得していることが分かった。

これを用いて、指定の slabclass から item の allocation を行う。

ただその前に、入力から指定した slabclass の id を取得してこなくてはならない。  
今回は *safe_stroul* という関数を用いて、指定した slabclass の id を取得することにした。

その結果、以下のようなプログラムになった。

```
/* 指定の slabclass への alloc を行う */
    if(!(safe_strtoul(tokens[2].value, &id))){
        out_string(c, "CLIENT_ERROR bad command line format");
        return;
    }
    fprintf(stderr, "In move command: id: %d\n", id);

    // 指定の slabclass から item のポインタをもらってくる
    it = do_item_alloc_pull(pre_it->nbytes, id);
    fprintf(stderr, "In move command: it->nbytes: %d\n", it->nbytes);
```

## 元の item のデータをコピーしてくる
次に、元の item のデータをコピーして来ようと思う  
一番簡単そうなのは item のサイズの分、元の item から memcpy を使ってコピーしてくること。

なのだが…、なぜか知らんが item 構造体の中ではなく、隣に key やら value やらが格納されている…。

面倒なのだが、そいつらはそれぞれコピーしてくことにした。  
ついでに、suffix とかいうやつあるんだが、これはなんだ?? 接尾辞だから、item の終わり部分とか??

```
 /* item の内容のコピー */
    memcpy(it, pre_it, sizeof(item));
    memcpy(ITEM_key(it), key, nkey);
    strcpy(ITEM_data(it), ITEM_data(pre_it));
    strcpy(ITEM_suffix(it), ITEM_suffix(pre_it));
```


## 元の item を削除する
thread.c にあったやつをそのまま使用  
おそらく片方が LRU や ハッシュから削除するやつで、
もう片方が slab から排除するやつ…だと思う。

```
 /* LRU とハッシュからの削除 */
    item_unlink(pre_it);
    item_remove(pre_it);
```

## 新規の item を LRU と hash にぶち込む!
最後に新規アイテムを LRU と hash にぶち込む  
thread.c に同じようにいい感じの *item_link* 関数があるのでそれを使用する。

ただしただし、slabclass の id を書き換えないと LRU がおかしくなってしまうので書き換えは忘れないように
```
// 指定の lru に代入し、hash に追加
    it->slabs_clsid = id | HOT_LRU; // 適当に HOT LRU に入れてみる
    fprintf(stderr, "In move command: it->slabs_clsid: %d\n", it->slabs_clsid);
    item_link(it);
```

## 実行結果
なんやかんやで move 命令が作成出来たっぽいので実験してみる。
まずは client 側で set 命令
```
set hoge 0 0 4
fuga
STORED
show
show received
```
するとサーバー側では
```
slab class   1: chunk size        96 perslab   10922    slabs 1
COLD: 
        key: hoge,      value: fuga

slab class   2: chunk size       120 perslab    8738    slabs 0
slab class   3: chunk size       152 perslab    6898    slabs 0
```

予想通り、slab class 1番に挿入されている。

ということで、2 番に移してみる
```
show received
move hoge 2
move received
show
show received
```
どうなる???
```
slab class   1: chunk size        96 perslab   10922    slabs 1
slab class   2: chunk size       120 perslab    8738    slabs 1
COLD: 
        key: hoge,      value: fuga

slab class   3: chunk size       152 perslab    6898    slabs 0
```
きた!!!!!!  
slab class 2 番に移動完了!
ちなみに get も普通に利用可能っぽい!!
```
get hoge
VALUE hoge 0 4
fuga
END
```