---
layout: post
title: The Curious Nature of the Salesforce Boolean
---

The Boolean can be among the simplest data types: it is either true or false, full stop, no complications.

On the Salesforce platform, this couldn't be further than the truth.

Consider the following subtle bug:

    public class BooleanPropertyCheck {
        public Boolean myProperty { get; set; }
        
        // ... more code here ...
    }
    
    @isTest
    public class BooleanPropertyCheckTest {
        @isTest
        public static void testTheThing() {
            BooleanPropertyCheck bpc = new BooleanPropertyCheck();

            System.assert(bpc.myProperty); 
        }
    }
    
That ought to be fine - we're `assert`ing a Boolean value, which is what asserts are for. But here, it's not fine. Instead, we get a `NullPointerException`. This reveals three things, in ascending order:

  1. The value of `myProperty` is `null` when not uninitialized, which is *not* equivalent to `false` in Apex;
  1. The `System.assert()` method throws an exception when passed `null`;
  1. Boolean variables in Apex are actually tri-valued: they can be `true`, `false`, or `null`, and they default to `null`.
  
That's all well and good, at a certain level; there's nothing all that wrong with tri-valued Booleans, although it's arguable that the behavior of `System.assert()` is poorly designed. What's far more confusing is that there are other layers of the Salesforce platform where Booleans are *not* treated as tri-valued.
