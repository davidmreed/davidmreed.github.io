---
layout: post
title: Free Events in Click & Pledge
---

Click & Pledge nicely supports online registration for both free and paid/ticketed events. However, one point of confusion for
both users and staff is that a "$0.00" line item is displayed on the registration page even for free events, and for free events using the named event model, a "Payment Information" section is displayed on the second page of the registration UI.

Fortunately, C&P offers the ability to include custom CSS in the template designer. Some [simple CSS](https://gist.github.com/davidmreed/61a46d18aee96b45a62f21fcfe27c12a) suppresses these payment-related
elements, granting a more streamlined free registration experience for both named and anonymous events. The effect isn't perfect, but until Click & Pledge permits deeper modification of its registration component, I'm satisfied with the results. Enhancements are welcome!
