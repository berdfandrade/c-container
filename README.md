# Linux containers in 500 lines of code 

Following up the tutorial : https://blog.lizzie.io/linux-containers-in-500-loc.html

"I've used Linux containers directly and indirectly for years, but I wanted to become more familiar with them. So I wrote some code. This used to be 500 lines of code, I swear, but I've revised it some since publishing; I've ended up with about 70 lines more.

I wanted specifically to find a minimal set of restrictions to run untrusted code. This isn't how you should approach containers on anything with any exposure: you should restrict everything you can. But I think it's important to know which permissions are categorically unsafe! I've tried to back up things I'm saying with links to code or people I trust, but I'd love to know if I missed anything.

This is a noweb-style piece of literate code. References named `<<x>>` will be expanded to the code block named x. You can find the tangled source here. This document is an orgmode document, you can find its source here. This document and this code are licensed under the GPLv3; you can find its source here."

