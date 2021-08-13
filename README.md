# Add move to memcached
このリポジトリでは、slab の中身を出力する命令を追加する  
今回は「move」という命令を新たに追加し、それを受け取った場合に指定された item を指定された slabclass に移送させることにした
***
## 基本方針
* 「move {keyの名称} {移送先の slabclass の id}」という形で client 側からの命令を受け取る
* *item_get* 関数で指定の item のアドレスを入手する
* *item_alloc* 関数で item を指定の slab に入れてあげる
* なんかの方法で元の slab のデータを free する (*slabs_free*?)

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
## 