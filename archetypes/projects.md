+++
title = "{{ replace .Name "-" " " | title }}"
tags = [ ]
date = {{ .Date }}
draft = true

[[project]]
status = "supported"

[[repo]]
type = "github"
spec = "hatf0/{{ .Name }}"
+++
{{< project >}}
{{< repo >}}
