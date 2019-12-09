---
title: "Querying Yaml and Json"
date: 2019-12-09T23:08:35+13:00
draft: true
tags: [yaml, json, yq, jq, kubernetes, transform]
---

Many application and infrastructure configurations are stored as YAML or JSON as a good balance between human readability and machine disambiguity. 

With YAML and JSON documents being so prevalent these days, it is a good idea to have some tricks up your sleeve to quickly sift through them, extracting the information you need, or transforming the structure.

My favourite tools for dealing with these configurations are:

- [`yq`][yq]: a wrapper around jq.
- [`jq`][jq]: json query.

Anything you can do with `jq` you can do with `yq`, with the bonus of being able to output yaml too (with the `-y` flag).

For example:

```sh
# JSON output by default
echo '{"a":{"b": [1,2,3],"c": [4,5,6]}}' | yq .
{
  "a": {
    "b": [
      1,
      2,
      3
    ],
    "c": [
      4,
      5,
      6
    ]
  }
}

# YAML output
echo '{"a":{"b": [1,2,3],"c": [4,5,6]}}' | yq -y .
a:
  b:
  - 1
  - 2
  - 3
  c:
  - 4
  - 5
  - 6
```

I have made the examples portable, so you should be able to run them yourself by just copying, pasting, and modifying to suit your own curiosity.

## Quick install

These tools can be installed by your favourite package manager, though for portabilities sake (since I use both MacOS and Ubuntu), I will just use containerised versions of the tools.

> **todo**: build and release the docker images

```sh
cat >> ~/.bashrc <<EOF
# docker aliases for tools
alias yq='docker run --rm -it nicklarsennz/yq'
alias jq='docker run --rm -it nicklarsennz/yq jq'
EOF
```

Throughout this guide, I'll focus on YAML and `yq`, but the same essentially applied to JSON and `jq`.

## Extracting parts of the document

> **todo**: simple example walking down the tree. Modify the narrative below to build off this.

Suppose you are running Apache Kafka in a Kubernetes StatefulSet, and want to check the images used in a running pod (for example, to see if a pod has taken the newly specified image).

You might run `kubectl get pod local-kafka-0 -o yaml` which produces a [Pod spec like this][pod-manifest].

_**Note**: In place of `kubectl` I'll use `curl` so you can copy/paste and have a go yourself._

```sh
SPEC="https://nicklarsennz.github.io/kubernetes/kafka-pod.yml"
curl "$SPEC" | yq -r '.spec.containers[] | .image'
```

The above command resulted in:

```txt
solsson/kafka-prometheus-jmx-exporter@sha256:a23062396cd5af1acdf76512632c20ea6be76885dfc20cd9ff40fb23846557e8
confluentinc/cp-kafka:5.0.1
```

So, we just got a list of images defined in the YAML file. The output isn't YAML, nor JSON because I supplied the `-r` flag which means print it out raw.

The query `.spec.containers[] | .image` can be broken down into:
- Return each entry (denoted by `[]`) of the list under `spec.containers`.
- Pipe (`|`) the output for further operation.
- Return the value of the `image` key (this is the last operation, so the results get printed out)

\
_Try it yourself, and see what happens if you remove the `| .image` from the query._

## Piping to builtin functions

> **todo**: pipe to upper case

## Arithmetic operations

> **todo**: double each number

## Conditionally extracting parts of the document

> **todo**: get all images from a manifest where ...

> **todo**: return the value of an annotation when the annotation key is ...

## Transformations

> **todo**: return a new list of container names to image

## Using input arguments

> **todo**: supply a number as an argument to multiply add to container port numbers by.
> **todo**: replace latest with a given image tag

## Quiz

> **todo**: think of a non-trivial manipulation to the structure

> **todo**: decide whether to keep this section

[yq]: https://yq.readthedocs.io/en/latest/
[jq]: https://stedolan.github.io/jq/download/
[bat]: https://github.com/sharkdp/bat
[pod-manifest]: /kubernetes/kafka-pod.yml