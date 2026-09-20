**core finding**: some functionality is obsolete with the modern UI. but functionality should still be supported - players should decicde which UI they want to use these mods with.

some mods (the ones marked obsolete here) should only be usable with old UI - so this merits a "feature flag" style implementation, where the mod's loaded assets depend on what UI the player uses in their settings.

# UAT 1: qudux convo option: works
initially no text
![alt text](image.png)

but present after first trade and exiting out of trade window
![alt text](image-1.png)

![alt text](image-2.png)

# UAT 2: Obsolete: tilemaker for NPC portraits and items
screenshots below aim to prove that functionality is obsolete with modern-ui enabled

confirmed working though with old UI

## npc icons
![alt text](image-10.png)
![alt text](image-11.png)

## item icons
![alt text](image-12.png)
![alt text](image-13.png)

# UAT 3.1: Quick Pickup Menu works
![alt text](image-14.png)
![alt text](image-15.png)
![alt text](image-16.png)

# UAT 3.2: Items on the ground - QuickPickUp doesnt work
![alt text](image-17.png)
![alt text](image-18.png)
![alt text](image-19.png)
![alt text](image-20.png)
also doesnt work in a chest
![alt text](image-21.png)

most likely not compatabile with UI overhaul

# UAT 3.3: autolooting
autoloot doesnt work correctly, even with old UI on- probably something new to how items are interpreted in the game.

![alt text](image-22.png)
![alt text](image-23.png)

"disable auto-pickup" in context menu toggle works.
![alt text](image-41.png)
![alt text](image-42.png)

- before re-implementing this feature, check if obsolete via native functionality

# UAT 4.1: qudux menu for recipes is obsolete
it works, but the native menu of modern ui makes it obsolute.

native menu:
![alt text](image-25.png)

# UAT 5 OK: restock countdown in conversations works 
4-5 days
![alt text](image-26.png)

1-2 days after waiting
![alt text](image-27.png)

# UAT 6: sprite menu OK
note: could use overhaul for modern UI

![alt text](image-28.png)

changing sprites is OK
![alt text](image-30.png)

colors OK
![alt text](image-29.png)

-- issue: if a callout (presumably modern ui issue) renders on top of sprite menu, it gets stuck and you cant exit anymore (esc/5/etc stops working - only fix is to restart the game)
![alt text](image-31.png)

# UAT 7 - quest giver locator
missing option, talked to multiple NPCs.
NPCs only show standard taext of "finx X / Y".
no guide to mehmet or argyve
![alt text](image-32.png)
![alt text](image-33.png)
![alt text](image-34.png)
![alt text](image-35.png)

# UAT 8 - scoreboard: showing old ui, gets stuck when modern-ui modal renders over it
![alt text](image-36.png)
![alt text](image-37.png)
![alt text](image-38.png)
![alt text](image-39.png)

unable to open "view quick keys" (?) on german keyboard.
works w/ english keyboard.

same issue as UAT 6: gets stuck when modal renders on top of it - cant exist anymore except game restart
![alt text](image-40.png)

# UAT 10: Autoloot generics (e.g. copper) works, but no modal
theres no modal window. probably tries to call old UI on top of modern UI, which conflicts?

autoloot works for copper
![alt text](image-43.png)

turned off autoloot for copper
![alt text](image-44.png)

autoloot-filter confirmed working for copper
![alt text](image-45.png)

# UAT 11.1: marking a legendary location
context menu item exists
- sidenote (not in scope for this): "mark location" is a single-set action that can be done over and over. its not a toggle
![alt text](image-46.png)

# UAT 11.2: journal is not written after "marking": not working
(all others are also empty)
![alt text](image-47.png)

# UAT 11.3: marking all heroes in zone: not working
![alt text](image-48.png)

"you havent noticed any legendary creature here"
![alt text](image-49.png)