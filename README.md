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