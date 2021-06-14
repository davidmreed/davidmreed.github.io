---
layout: post
title: "\"What's the best translation of...?\""
---

I've launched a new app, [Bibliothekai](https://bibliothekai.ktema.org/), to help answer this question.

I've been wanting to build this for many years, stemming from my time as a desk librarian hearing the question "What's the best translation of X?" all the time, and my delight in collecting unusual translations of Homer.

Bibliotheikai is my contribution towards an answer. Bibliothekai currently tracks over 900 translations of more than 275 ancient source texts. Coverage of extant translations is nearly complete on Homer (20th cent. and later), Herodotus, and Thucydides, and growing for authors from Plato to Virgil to Apollonius (of Rhodes and of Perga!) There's much, much more to do, though.

Bibliothekai makes it easy to sort and filter translations (like those of the [Iliad](https://bibliothekai.ktema.org/texts/2/)) for specific qualities and resources, to evaluate translations through professional and user reviews, and to compare selected translations with one another and with the original source text. For example, here's a side-by-side comparison of two prominent translations of the Iliad, [those of Richmond Lattimore and Robert Fagles](https://bibliothekai.ktema.org/texts/2/translations?trans=494&trans=77&hideOriginal=true), or one can compare [the Pamela Mensch translation of Herodotus with the Greek](https://bibliothekai.ktema.org/texts/75/translations?trans=48).

Bibliothekai is built on a Django backend, with a hybrid frontend using both Django templating and [open source Lightning Web Components](https://lwc.dev). I'll be writing more about how the site's been built in the coming weeks:

- Adding LWCs to a templated Django site running on Heroku.
- Creating your own LWC base components and using Bootstrap styling with open source LWCs.
- Building custom wire adapters to source data from Django Rest Framework and Graphene-Django for GraphQL.
- GraphQL integrations with Lightning Web Components.

Bibliothekai (which means "bookcases" in Ancient Greek) has been a fantastic opportunity to build something I've been passionate about for a long time, while teaching myself new technologies. I'm excited to share more about how it's built and the new features I'm working on.