+++
title = "My First RE Post"
date = 2026-04-03
draft = false
tags = ["reverse engineering", "x86", "malware"]
image = "/images/post-cover.png"   # optional cover image
+++

Some assembly I found interesting:

```x86asm
push ebp
mov  ebp, esp
sub  esp, 0x10
xor  eax, eax
```

Analysis goes here...