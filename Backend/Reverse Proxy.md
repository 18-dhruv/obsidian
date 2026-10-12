 ## A reverse proxy sits in front of your API and forward request to it, while clients only ever talks to proxy they never see or your actual api address or port. 

## config file setup

main  (the top of the file; not written as a block)
├── events { }
├── http { }
│   ├── upstream NAME { }
│   ├── map $SOURCE $TARGET { }
│   ├── geo $VARIABLE { }
│   ├── split_clients "STRING" $VARIABLE { }
│   ├── types { }
│   └── server { }
│       ├── if (CONDITION) { }
│       └── location [MODIFIER] PATH { }
│           ├── location ... { }        (can be nested)
│           ├── if (CONDITION) { }
│           └── limit_except METHOD { }
├── stream { }   (TCP/UDP proxying, not HTTP)
│   ├── upstream NAME { }
│   └── server { }
└── mail { }     (email proxying, rarely used)
    └── server { }



## main 
- user-> just we are setting of the
- worker processes >
