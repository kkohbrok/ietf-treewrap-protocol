---
title: "TreeWrap Protocol"
category: info

docname: draft-kohbrok-ietf-treewrap-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "IETF"
keyword:
 - encryption
 - forward secrecy
 - post-compromise security
venue:
  group: "IETF"
  type: "Internet Engineering Task Force"
  mail: ""
  arch: ""
  github: "kkohbrok/ietf-treewrap-protocol"
  latest: "https://kkohbrok.github.io/ietf-treewrap-protocol/draft-kohbrok-ietf-treewrap-protocol.html"

author:
 -
    fullname: Raphael Robert
    organization: Phoenix R&D
    email: ietf@raphaelrobert.com
 -
    fullname: Konrad Kohbrok
    organization: Phoenix R&D
    email: konrad.kohbrok@datashrine.de
 -
    fullname: Joël Alwen
    email: alwenjo@amazon.com
 -
    fullname: Marta Mularczyk
    email: mulmarta@amazon.com

normative:

informative:

--- abstract

TreeWrap is a protocol to distribute and manage keys. Keys can be added, removed
(to provide forward secrecy) and updated (to provide post-compromise security).
TreeWrap key material can be stored and distributed efficiently through
untrusted infrastructure.

--- middle

# Introduction

TODO Introduction
