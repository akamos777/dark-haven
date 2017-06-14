## 5) Advanced List Operations

The above section covers basic comparisons. There are a few more powerful features as well, but - as anyone familiar with mathematical   sets will know - things begin to get a bit fiddly. So this section comes with an 'advanced' warning.

A lot of the features in this section won't be necessary for most games.

### Comparing lists

We can compare lists less than exactly using `>`, `<`, `>=` and `<=`. Be warned! The definitions we use are not exactly standard fare. They are based on comparing the numerical value of the elements in the lists being tested.

#### "Distinctly bigger than"

`LIST_A > LIST_B` means "the smallest value in A is bigger than the largest values in B": in other words, if put on a number line, the entirety of A is to the right of the entirety of B. `<` does the same in reverse.

#### "Definitely never smaller than"

`LIST_A >= LIST_B` means - take a deep breath now - "the smallest value in A is at least the smallest value in B, and the largest value in A is at least the largest value in B". That is, if drawn on a number line, the entirety of A is either above B or overlaps with it, but B does not extend higher than A.

Note that `LIST_A > LIST_B` implies `LIST_A != LIST_B`, and `LIST_A >= LIST_B` allows `LIST_A == LIST_B` but precludes `LIST_A < LIST_B`, as you might hope.

#### Health warning!

`LIST_A >= LIST_B` is *not* the same as `LIST_A > LIST_B or LIST_A == LIST_B`.

The moral is, don't use these unless you have a clear picture in your mind.

### Inverting lists

A list can be "inverted", which is the equivalent of going through the accommodation in/out name-board and flipping every switch to the opposite of what it was before.

	LIST GuardsOnDuty = (Smith), (Jones), Carter, Braithwaite

	=== function changingOfTheGuard
		~ GuardsOnDuty = LIST_INVERT(GuardsOnDuty)


Note that `LIST_INVERT` on an empty list will return a null value, if the game doesn't have enough context to know what invert. If you need to handle that case, it's safest to do it by hand:

	=== function changingOfTheGuard
		{!GuardsOnDuty: // "is GuardsOnDuty empty right now?"
			~ GuardsOnDuty = LIST_ALL(Smith)
		- else:
			~ GuardsOnDuty = LIST_INVERT(GuardsOnDuty)
		}

#### Footnote

The syntax for inversion was originally `~ list` but we changed it because otherwise the line

	~ list = ~ list

was not only functional, but actually caused list to invert itself, which seemed excessively perverse.

### Intersecting lists

The `has` or `?` operator is, somewhat more formally, the "are you a subset of me" operator, ⊇, which includes the sets being equal, but which doesn't include if the larger set doesn't entirely contain the smaller set.

To test for "some overlap" between lists, we use the overlap operator, `^`, to get the *intersection*.

	LIST CoreValues = strength, courage, compassion, greed, nepotism, self_belief, delusions_of_godhood
	VAR desiredValues = (strength, courage, compassion, self_belief )
	VAR actualValues =  ( greed, nepotism, self_belief, delusions_of_godhood )

	{desiredValues ^ actualValues} // prints "self_belief"

The result is a new list, so you can test it:

	{desiredValues ^ actualValues: The new president has at least one desirable quality.}

	{LIST_COUNT(desiredValues ^ actualValues) == 1: Correction, the new president has only one desirable quality. {desiredValues ^ actualValues == self_belief: It's the scary one.}}




## 6) Multi-list Lists


So far, all of our examples have included one large simplification, again - that the values in a list variable have to all be from the same list family. But they don't.

This allows us to use lists - which have so far played the role of state-machines and flag-trackers - to also act as general properties, which is useful for world modelling.

This is our inception moment. The results are powerful, but also more like "real code" than anything that's come before.

### Lists to track objects

For instance, we might define:

	LIST Characters = Alfred, Batman, Robin
	LIST Props = champagne_glass, newspaper

	VAR BallroomContents = (Alfred, Batman, newspaper)
	VAR HallwayContents = (Robin, champagne_glass)

We could then describe the contents of any room by testing its state:

	=== function describe_room(roomState)
		{ roomState ? Alfred: Alfred is here, standing quietly in a corner. } { roomState ? Batman: Batman's presence dominates all. } { roomState ? Robin: Robin is all but forgotten. }
		<> { roomState ? champagne_glass: A champagne glass lies discarded on the floor. } { roomState ? newspaper: On one table, a headline blares out WHO IS THE BATMAN? AND *WHO* IS HIS BARELY-REMEMBER ASSISTANT? }

So then:

	{ describe_room(BallroomContents) }

produces:

	Alfred is here, standing quietly in a corner. Batman's presence dominates all.

	On one table, a headline blares out WHO IS THE BATMAN? AND *WHO* IS HIS BARELY-REMEMBER ASSISTANT?

While:

	{ describe_room(HallwayContents) }

gives:

	Robin is all but forgotten.

	A champagne glass lies discarded on the floor.

And we could have options based on combinations of things:

	*	{ currentRoomState ? (Batman, Alfred) } [Talk to Alfred and Batman]
		'Say, do you two know each other?'

### Lists to track multiple states

We can model devices with multiple states. Back to the kettle again...

	LIST OnOff = on, off
	LIST HotCold = cold, warm, hot

	VAR kettleState = off, cold

	=== function turnOnKettle() ===
	{ kettleState ? hot:
		You turn on the kettle, but it immediately flips off again.
	- else:
		The water in the kettle begins to heat up.
		~ kettleState -= off
		~ kettleState += on
		// note we avoid "=" as it'll remove all existing states
	}

	=== function can_make_tea() ===
		~ return kettleState ? (hot, off)

These mixed states can make changing state a bit trickier, as the off/on above demonstrates, so the following helper function can be useful.

 	=== function changeStateTo(ref stateVariable, stateToReach)
 		// remove all states of this type
 		~ stateVariable -= LIST_ALL(stateToReach)
 		// put back the state we want
 		~ stateVariable += stateToReach

 which enables code like:

 	~ changeState(kettleState, on)
 	~ changeState(kettleState, warm)




#### How does this affect queries?

The queries given above mostly generalise nicely to multi-valued lists

    LIST Letters = a,b,c
    LIST Numbers = one, two, three

    VAR mixedList = (a, three, c)

	{LIST_ALL(mixedList)}   // a, one, b, two, c, three
    {LIST_COUNT(mixedList)} // 3
    {LIST_MIN(mixedList)}   // a
    {LIST_MAX(mixedList)}   // three or c, albeit unpredictably

    {mixedList ? (a,b) }        // false
    {mixedList ^ LIST_ALL(a)}   // a, c

    { mixedList >= (one, a) }   // true
    { mixedList < (three) }     // false

	{ LIST_INVERT(mixedList) }            // one, b, two



## 7) Long example: crime scene

Finally, here's a long example, demonstrating a lot of ideas from this section in action. You might want to try playing it before reading through to better understand the various moving parts.

	-> murder_scene

	//
	// 	System: items can have various states
	//	Some are general, some specific to particular items
	//

	LIST OffOn = off, on
	LIST SeenUnseen = unseen, seen

	LIST GlassState = (none), steamed, steam_gone
	LIST BedState = (made_up), covers_shifted, covers_off, stain_visible

	//
	// System: inventory
	//

	LIST Inventory = (none), cane, knife

	=== function get(x)
	    ~ move_to_supporter(x, held)
	    ~ Inventory += x

	//
	// System: positioning things
	// Items can be put in and on places
	//

	LIST Supporters = on_desk, on_floor, on_bed, under_bed, held, with_joe

	=== function move_to_supporter(ref item_state, new_supporter) ===
	    { Inventory ? item_state && new_supporter != held:
	        ~ Inventory -= item_state
	    }
	    ~ item_state -= LIST_ALL(Supporters)
	    ~ item_state += new_supporter

	//
	// System: Incremental knowledge.
	// Each list is a chains of facts. Each fact supercedes the fact before it.
	//


	LIST BedKnowledge = (none), neatly_made, crumpled_duvet, hastily_remade, body_on_bed, murdered_in_bed, murdered_while_asleep
	LIST KnifeKnowledge = (none), prints_on_knife, joe_seen_prints_on_knife,joe_wants_better_prints, joe_got_better_prints
	LIST WindowKnowledge = (none), steam_on_glass, fingerprints_on_glass, fingerprints_on_glass_match_knife

	VAR knowledgeState = ()

	=== function learn(x) ===
		// learn this fact
	    ~ knowledgeState += x

	=== function learnt(x) ===
		// have you learnt this fact, or indeed a stronger one
	    ~ return highest_state_for_set_of_state(x) >= x

	=== function between(x, y) ===
		// are you between two ideas? Not necessarily in the same knowledge tree.
	    ~ return learnt(x) && not learnt(y)

	=== function think(x) ===
		// is this your current "strongest" idea in this knowledge set?
	    ~ return highest_state_for_set_of_state(x) == x

	=== function highest_state_for_set_of_state(x) ===
	    ~ return LIST_MAX(knowledgeState ^ LIST_ALL(x))

	=== function did_learn(x) ===
		//	did you learn this particular fact?
	    ~ return knowledgeState ? x

	//
	// Set up the scene
	//

	VAR bedroomLightState = (off, on_desk)
	VAR knifeState = (under_bed)

	//
	// Content
	//

	=== murder_scene ===
	    The bedroom. This is where it happened. Now to look for clues.
	- (top)
	    { bedroomLightState ? seen:     <- seen_light  }
	    <- compare_prints(-> top)

	    *   (dobed) [The bed...]
	        The bed was low to the ground, but not so low something might not roll underneath. It was still neatly made.
	        ~ learn(neatly_made)
	        - - (bedhub)
	        * *     [Lift the bedcover]
	                I lifted back the bedcover. The duvet underneath was crumpled.
	                ~ learn(crumpled_duvet)
	                ~ BedState = covers_shifted
	        * *     (uncover) {learnt(crumpled_duvet)}
	                [Remove the cover]
	                Careful not to disturb anything beneath, I removed the cover entirely. The duvet below was rumpled.
	                Not the work of the maid, who was conscientious to a point. Clearly this had been thrown on in a hurry.
	                ~ learn(hastily_remade)
	                ~ BedState = covers_off
	        * *     (duvet) {BedState == covers_off} [ Pull back the duvet ]
	                I pulled back the duvet. Beneath it was a sheet, sticky with blood.
	                ~ BedState = stain_visible
	                ~ learn(body_on_bed)
	                Either the body had been moved here before being dragged to the floor - or this is was where the murder had taken place.
	        * *     {!(BedState ? made_up)} [ Remake the bed ]
	                Carefully, I pulled the bedsheets back into place, trying to make it seem undisturbed.
	                ~ BedState = made_up
	        * *     [Test the bed]
	                I pushed the bed with spread fingers. It creaked a little, but not so much as to be obnoxious.
	        * *     (darkunder) [Look under the bed]
	                Lying down, I peered under the bed, but could make nothing out.

	        * *     {TURNS_SINCE(-> dobed) > 1} [Something else?]
	                I took a step back from the bed and looked around.
	                -> top
	        - -     -> bedhub

	    *   {darkunder && bedroomLightState ? on_floor && bedroomLightState ? on}
	        [ Look under the bed ]
	        I peered under the bed. Something glinted back at me.
	        - - (reaching)
	        * *     [ Reach for it ]
	                I fished with one arm under the bed, but whatever it was, it had been kicked far enough back that I couldn't get my fingers on it.
	                -> reaching
	        * *     {Inventory ? cane} [Knock it with the cane]
	                -> knock_with_cane

	        * *     {reaching > 1 } [ Stand up ]
	                I stood up once more, and brushed my coat down.
	                -> top

	    *   (knock_with_cane) {reaching && TURNS_SINCE(-> reaching) >= 4 &&  Inventory ? cane } [Use the cane to reach under the bed ]
	        Positioning the cane above the carpet, I gave the glinting thing a sharp tap. It slid out from the under the foot of the bed.
	        ~ move_to_supporter( knifeState, on_floor )
	        * *     (standup) [Stand up]
	                Satisfied, I stood up, and saw I had knocked free a bloodied knife.
	                -> top
	        * *     [Look under the bed once more]
	                Moving the cane aside, I looked under the bed once more, but there was nothing more there.
	                -> standup

	    *   {knifeState ? on_floor} [Pick up the knife]
	        Careful not to touch the handle, I lifted the blade from the carpet.
	        ~ get(knife)

	    *   {Inventory ? knife} [Look at the knife]
	        The blood was dry enough. Dry enough to show up partial prints on the hilt!
	        ~ learn(prints_on_knife)

	    *   [   The desk... ]
	        I turned my attention to the desk. A lamp sat in one corner, a neat, empty in-tray in the other. There was nothing else out.
	        Leaning against the desk was a wooden cane.
	        ~ bedroomLightState += seen
	        - - (deskstate)
	        * *     (pickup_cane) {Inventory !? cane}    [Pick up the cane ]
	                ~ get(cane)
	                I picked up the wooden cane. It was heavy, and unmarked.

	        * *    { bedroomLightState !? on } [Turn on the lamp]
	                -> operate_lamp ->
	        * *     [Look at the in-tray ]
	                I regarded the in-tray, but there was nothing to be seen. Either the victim's papers were taken, or his line of work had seriously dried up. Or the in-tray was all for show.
	        + +     (open)  {open < 3} [Open a drawer]
	                I tried {a drawer at random|another drawer|a third drawer}. {Locked|Also locked|Unsurprisingly, locked as well}.

	        * *     {deskstate >= 2} [Something else?]
	                I took a step away from the desk once more.
	                -> top
	        - -     -> deskstate

	    *     {(Inventory ? cane) && TURNS_SINCE(-> deskstate) <= 2} [Swoosh the cane]
	        I was still holding the cane: I gave it an experimental swoosh. It was heavy indeed, though not heavy enough to be used as a bludgeon.
	        But it might have been useful in self-defence. Why hadn't the victim reached for it? Knocked it over?

	    *   [The window...]
	        I went over to the window and peered out. A dismal view of the little brook that ran down beside the house.
	        - - (window_opts)
	        <- compare_prints(-> window_opts)
	        * *     (downy) [Look down at the brook]
	                { GlassState ? steamed:
	                    Through the steamed glass I couldn't see the brook. -> see_prints_on_glass -> window_opts
	                }
	                I watched the little stream rush past for a while. The house probably had damp but otherwise, it told me nothing.
	        * *     (greasy) [Look at the glass]
	                { GlassState ? steamed: -> downy }
	                The glass in the window was greasy. No one had cleaned it in a while, inside or out.
	        * *     { GlassState ? steamed && not see_prints_on_glass && downy && greasy }
	                [ Look at the steam ]
	                A cold day outside. Natural my breath should steam. -> see_prints_on_glass ->
	        + +     {GlassState ? steam_gone} [ Breathe on the glass ]
	                I breathed gently on the glass once more. {learnt(fingerprints_on_glass): The fingerprints reappeared. }
	                ~ GlassState = steamed

	        + +     [Something else?]
	                { window_opts < 2 || learnt(fingerprints_on_glass) || GlassState ? steamed:
	                    I looked away from the dreary glass.
	                    {GlassState ? steamed:
	                        ~ GlassState = steam_gone
	                        <> The steam from my breath faded.
	                    }
	                    -> top
	                }
	                I leant back from the glass. My breath had steamed up the pane a little.
	               ~ GlassState = steamed
	        - -     -> window_opts



	    *   {top >= 5} [Leave the room]
	        I'd seen enough. I {bedroomLightState ? on:switched off the lamp, then} turned and left the room.
	        -> joe_in_hall
	    -   -> top

	= see_prints_on_glass
	    ~ learn(fingerprints_on_glass)
	    {But I could see a few fingerprints, as though someone had leant their palm against it.|The fingerprints were quite clear and well-formed.} They faded as I watched.
	    ~ GlassState = steam_gone
	    ->->

	= compare_prints (-> backto)
	    *   {learnt(fingerprints_on_glass) && learnt(prints_on_knife) && !learnt(fingerprints_on_glass_match_knife)} [Compare the prints on the knife and the window ]
	        Holding the bloodied knife near the window, I breathed to bring out the prints once more, and compared them as best I could.
	        Hardly scientific, but they seemed very similar - very similiar indeed.
	        ~ learn(fingerprints_on_glass_match_knife)
	        -> backto

	= operate_lamp
	    I flicked the light switch.
	    { bedroomLightState ? on:
	        <> The bulb fell dark.
	        ~ bedroomLightState += off
	        ~ bedroomLightState -= on
	    - else:
	        { bedroomLightState ? on_floor: <> A little light spilled under the bed.} { bedroomLightState ? on_desk : <> The light gleamed on the polished tabletop. }
	        ~ bedroomLightState -= off
	        ~ bedroomLightState += on
	    }
	    ->->

	= seen_light
	    *   {!(bedroomLightState ? on)} [ Turn on lamp ]
	        -> operate_lamp ->

	    *   { !(bedroomLightState ? on_bed)  && BedState ? stain_visible }
	        [ Move the light to the bed ]
	        ~ move_to_supporter(bedroomLightState, on_bed)
	        I moved the light over to the bloodstain and peered closely at it. It had soaked deeply into the fibres of the cotton sheet.
	        There was no doubt about it. This was where the blow had been struck.
	        ~ learn(murdered_in_bed)

	    *   { !(bedroomLightState ? on_desk) } {TURNS_SINCE(-> floorit) >= 2 }
	        [ Move the light back to the desk ]
	        ~ move_to_supporter(bedroomLightState, on_desk)
	        I moved the light back to the desk, setting it down where it had originally been.
	    *   (floorit) { !(bedroomLightState ? on_floor) && darkunder }
	        [Move the light to the floor ]
	        ~ move_to_supporter(bedroomLightState, on_floor)
	        I picked the light up and set it down on the floor.
	    -   -> top

	=== joe_in_hall
	    My police contact, Joe, was waiting in the hall. 'So?' he demanded. 'Did you find anything interesting?'
	- (found)
	    *   {found == 1} 'Nothing.'
	        He shrugged. 'Shame.'
	        -> done
	    *   { Inventory ? knife } 'I found the murder weapon.'
	        'Good going!' Joe replied with a grin. 'We thought the murderer had gotten rid of it. I'll bag that for you now.'
	        ~ move_to_supporter(knifeState, with_joe)
	    *   {learnt(prints_on_knife)} { knifeState ? with_joe }
	        'There are prints on the blade[.'],' I told him.
	        He regarded them carefully.
	        'Hrm. Not very complete. It'll be hard to get a match from these.'
	        ~ learn(joe_seen_prints_on_knife)
	    *   { learnt(fingerprints_on_glass_match_knife) && learnt(joe_seen_prints_on_knife) }
	        'They match a set of prints on the window, too.'
	        'Anyone could have touched the window,' Joe replied thoughtfully. 'But if they're more complete, they should help us get a decent match!'
	        ~ learn(joe_wants_better_prints)
	    *   { between(body_on_bed, murdered_in_bed)}
	        'The body was moved to the bed at some point[.'],' I told him. 'And then moved back to the floor.'
	        'Why?'
	        * *     'I don't know.'
	                Joe nods. 'All right.'
	        * *     'Perhaps to get something from the floor?'
	                'You wouldn't move a whole body for that.'
	        * *     'Perhaps he was killed in bed.'
	                'It's just speculation at this point,' Joe remarks.
	    *   { learnt(murdered_in_bed) }
	        'The victim was murdered in bed, and then the body was moved to the floor.'
	        'Why?'
	        * *     'I don't know.'
	                Joe nods. 'All right, then.'
	        * *     'Perhaps the murderer wanted to mislead us.'
	                'How so?'
	            * * *   'They wanted us to think the victim was awake[.'], I replied thoughtfully. 'That they were meeting their attacker, rather than being stabbed in their sleep.'
	            * * *   'They wanted us to think there was some kind of struggle[.'],' I replied. 'That the victim wasn't simply stabbed in their sleep.'
	            - - -   'But if they were killed in bed, that's most likely what happened. Stabbed, while sleeping.'
	                    ~ learn(murdered_while_asleep)
	        * *     'Perhaps the murderer hoped to clean up the scene.'
	                'But they were disturbed? It's possible.'


	    *   { found > 1} 'That's it.'
	        'All right. It's a start,' Joe replied.
	        -> done
	    -   -> found
	-   (done)
	    {
	    - between(joe_wants_better_prints, joe_got_better_prints):
	        ~ learn(joe_got_better_prints)
	        <>  He moved for the door.  'I'll get those prints from the window now.'
	    - learnt(joe_seen_prints_on_knife):
	        <> 'I'll run those prints as best I can.'
	    - else:
	        <> 'Not much to go on.'
	    }
	    -> END

## 8) Summary

To summarise a difficult section, **ink**'s list construction provides:

### Flags
* 	Each list entry is an event
* 	Use `+=` to mark an event as having occurred
*  	Test using `?` and `!?`

Example:

	LIST GameEvents = foundSword, openedCasket, metGorgon
	{ GameEvents ? openedCasket }
	{ GameEvents ? (foundSword, metGorgon) }
	~ GameEvents += metGorgon

### State machines
* 	Each list entry is a state
*  Use `=` to set the state; `++` and `--` to step forward or backward
*  Test using `==`, `>` etc

Example:

	LIST PancakeState = ingredients_gathered, batter_mix, pan_hot, pancakes_tossed, ready_to_eat
	{ PancakeState == batter_mix }
	{ PancakeState < ready_to_eat }
	~ PancakeState++

### Properties
*	Each list is a different property, with values for the states that property can take (on or off, lit or unlit, etc)
* 	Change state by removing the old state, then adding in the new
*  Test using `?` and `!?`

Example:

	LIST OnOffState = on, off
	LIST ChargeState = uncharged, charging, charged

	VAR PhoneState = (off, uncharged)

	*	{PhoneState !? uncharged } [Plug in phone]
		~ PhoneState -= LIST_ALL(ChargeState)
		~ PhoneState += charging
		You plug the phone into charge.
	*	{ PhoneState ? (on, charged) } [ Call my mother ]
