+++
title = "Hugo Tests"
date = 2025-11-22T19:55:23+01:00
images = []
tags = ["test"]
menus = ['posts']
categories = ["coding"]
draft = false
+++

Hello :wave:

This is my first test post to test language rendering for Go and Bash and how to use pictures.

Something written in Go with highlights
```go {linenos=inline hl_lines=[3,"6-8"] style=emacs}
package main

import "fmt"

func main() {
    for i := 0; i < 3; i++ {
        fmt.Println("Value of i:", i)
    }
}
```

Another thing written in Go without highlights
```go {linenos=inline style=emacs}
package foo

func Bar() (string, error) {
     println("Hello foo.Bar!")
     return "foo", nil
}
```

Now a little bit of bash:
```bash {linenos=inline style=emacs}
function foo () {
     echo "Hello Bash!"
}

cat <<EOF
Hello World!
EOF

while [ 1 ]
do
        echo -n "."
        sleep 0.5
done
```

A bit of goat
```goat
.---.     .-.       .-.       .-.     .---.
| A +--->| 1 |<--->| 2 |<--->| 3 |<---+ B |
'---'     '-'       '+'       '+'     '---'
```

QR code:
{{< qr  level="high" scale=16 alt="QR code of skipper user documentation" >}}
https://opensource.zalando.com/skipper/
{{< /qr >}}


A bit of mathjax
\[
\begin{aligned}
KL(\hat{y} || y) &= \sum_{c=1}^{M}\hat{y}_c \log{\frac{\hat{y}_c}{y_c}} \\
JS(\hat{y} || y) &= \frac{1}{2}(KL(y||\frac{y+\hat{y}}{2}) + KL(\hat{y}||\frac{y+\hat{y}}{2}))
\end{aligned}
\]

A bit of inline mathjax \(a^*=x-b^*\) and \(\alpha \gt \beta\).

SVG with figure:
{{< figure
	src=perf.svg
	alt="A CPU flamegraph"
	link="https://github.com/brendangregg/FlameGraph"
	caption="CPU flamegraph"
	class="ma0 w-75"
>}}

PNG figure:
{{< figure
	src=skipper.png
	alt="Link to skipper as logo"
	link="https://opensource.zalando.com/skipper"
	caption="skipper logo"
>}}


## Tests ##

{{ $v1 := 6 }}
{{ $v2 := 7 }}
<p>The product of {{ $v1 }} and {{ $v2 }} is {{ mul $v1 $v2 }}.</p>

{{ $d = time.ParseDuration "3.5h2.5m1.5s" }}
{{ $d.Hours }}

## Resources ##


resources.Get (global assets)
{{ with resources.Get "perf.svg" }}
  <img src="{{ .RelPermalink }}" width="{{ .Width }}" height="{{ .Height }}" alt="">
{{ end }}


Resources.Get picture (per page) and try to render:

{{ $image := .Resources.Get "./skipper.png" }}
{{ with $image }}
    <img src="{{ .RelPermalink }}" width="{{ .Width }}" height="{{ .Height }}">
{{ end }}


{{ $image := .Resources.Get "skipper.png" }}
{{ with $image }}
![skipper logo markdown styl does not work]({{ .RelPermalink }})
{{ end }}


end
:) O:)
