---
title: “On Power Users”
date: 2023-02-11 12:00:00 -0500
categories: 
    - thoughts
tags: 
    - user experience 
    - applications
---


I wonder if I should let users self-identify as power users. 


In some settings menu or account profile screen I should include a toggle or checkbox that says “I consider myself a power user of this software”?


Letting the user self-identify as a power user will tell me who all my power users are and allow me to make design decisions for two different personas. 


By putting the setting in some relatively obscure menu with copy like “power-user” it will naturally filter out people that don’t dig deep into the menu screens and people that are unsure what that term means. This may sound harsh but if I want to hide things like “welcome text” or large chunks of context that are paramount for new or beginner users then I want to be sure I am only removing it for those that really don’t need it. 


It will add some complication to the UI code. Having to check “isPowerUser” all over the place might feel a bit clumsy but the alternative is a UX/UI that is at best slightly less optimal but at worst feels painfully remedial after learning it. For software that users are expected to use repetitively and with relatively short durations between sessions, I should anticipate that they will no longer need all the helpful copy and tooltips I have added to educate them on the software. 