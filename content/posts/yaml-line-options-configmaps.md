---
title: "YAML Line Options in Kubernetes ConfigMaps"
date: 2019-06-24T19:16:01-05:00
draft: false
author: "Joseph D. Marhee"
---

I was [recently asked about some formatting options that a peer saw in a Kubernetes manifest](https://www.facebook.com/groups/256621231871985/permalink/374169513450489/) for a ConfigMap resource. Because these manifests conform to the YAML spec, I was able to refer to a very specific resource for this purpose, but thought I'd run through it for the sake of completeness here.

Given a ConfigMap for data like:

```
data: 
  someKey: |-
    someValue
    someOtherValue
```

The question centered around the pipe `|` and the minus sign `-` characters (in combination), and what their purpose was in a block like the above.
I directed my peer to the YAML Spec for each of these items:

[Literal Style](https://yaml.org/spec/1.2/spec.html?fbclid=IwAR0iq51HUmx0Ig70HXFcDSddEh07xLVrMC4-Xo8TlT91ve478icChoks5q8#id2795688)

[Block Chomp Indicator](https://yaml.org/spec/1.2/spec.html?fbclid=IwAR0iq51HUmx0Ig70HXFcDSddEh07xLVrMC4-Xo8TlT91ve478icChoks5q8#id2794534)

but want to demonstrate the output as far as Kubernetes is concerned.
 
The `|` literal indicated that the following is a multi-line value, and that tabs, backslashes, etc. are considered content (everything at the proper level of indentation after the data key name), meaning that they will be rendered along with the rest of the block. 

The `-` chomping indicator instructs, when the YAML is rendered, that the final line break is removed (ending at the line the last character appears on, rather than a trailing line), rather than preserving (`+`) trailing lines.

So, with this in mind, let's look at an example, where we'll indicate we'd like to strip the trailing line breaks, and one where we'll preserve it:

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: test-configmap
data:
  chomp.strip: |-
    hello
    world



  chomp.keep: |+
    hello
    world





```

In `chomp.strip`, we'll use `-` to indicate we'd like the line breaks stripped after the last line with data, and in `chomp.keep`, we'll preserve it. 

To validate the formatting came out as expected, let's run `kubectl describe configmap/test-configmap`:

```
jmarhee@y2k-stepdad [19:47] ~ » kubectl describe configmap/test-configmap
Name:         test-configmap
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
chomp.keep:
----
hello
world






chomp.strip:
----
hello
world
Events:  <none>
```
and, as you see above, the formatting was preserved per that chomping indicator we set!

Kubernetes resources conform to the YAML spec (though the [reverse isn't always the case](https://medium.com/@jmarhee/example-of-yaml-generator-and-validator-in-python-5460505b5ad8)), so I recommend, if you found this useful, that you take a look at the rest of the spec (worth a read, at least once!) and see where else in your manifests you can benefit from knowing your formatting options.