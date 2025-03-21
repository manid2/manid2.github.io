Mani Kumar website
==================

This is the source code for my website [manid2.github.io][1]:

Start Hugo server
-----------------

```bash
hugo server

# serve drafts and future content
hugo server --buildDrafts --buildFuture
```

Print any page
--------------

```bash
pip install weasyprint
weasyprint {url} out.pdf
```

Use theme local updates
-----------------------

Add replace remote theme path with local path in Hugo modules.

```bash
$ tail go.mod
replace github.com/manid2/hugo-xterm => ../hugo-xterm
```

[1]: https://manid2.github.io
