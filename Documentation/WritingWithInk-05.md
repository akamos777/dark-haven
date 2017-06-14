# Part 5: Advanced State Tracking

Games with lots of interaction can get very complex, very quickly and the writer's job is often as much about maintaining continuity as it is about content.

This becomes particularly important if the game text is intended to model anything - whether it's a game of cards, the player's knowledge of the gameworld so far, or the state of the various light-switches in a house.

**ink** does not provide a full world-modelling system in the manner of a classic parser IF authoring language - there are no "objects", no concepts of "containment" or being "open" or "locked". However, it does provide a simple yet powerful system for tracking state-changes in a very flexible way, to enable writers to approximate world models where necessary.

#### Note: New feature alert!

This feature is very new to the language. That means we haven't begun to discover all the ways it might be used - but we're pretty sure it's going to be useful! So if you think of a clever usage we'd love to know!


## 1) Basic Lists

The basic unit of state-tracking is a list of states, defined using the `LIST` keyword. Note that a list is really nothing like a C# list (which is an array).

For instance, we might have:

	LIST kettleState = cold, boiling, recently_boiled

This line defines two things: firstly three new values - `cold`, `boiling` and `recently_boiled` - and secondly, a variable, called `kettleState`, to hold these states.

We can tell the list what value to take:

	~ kettleState = cold

We can change the value:

	*	[Turn on kettle]
		The kettle begins to bubble and boil.
		~ kettleState = boiling

We can query the value:

	*	[Touch the kettle]
		{ kettleState == cold:
			The kettle is cool to the touch.
		- else:
		 	The outside of the kettle is very warm!
		}

For convenience, we can give a list a value when it's defined using a bracket:

	LIST kettleState = cold, (boiling), recently_boiled
	// at the start of the game, this kettle is switched on. Edgy, huh?

...and if the notation for that looks a bit redundant, there's a reason for that coming up in a few subsections time.



## 2) Reusing Lists

The above example is fine for the kettle, but what if we have a pot on the stove as well? We can then define a list of states, but put them into variables - and as many variables as we want.

	LIST daysOfTheWeek = Monday, Tuesday, Wednesday, Thursday, Friday
	VAR today = Monday
	VAR tomorrow = Tuesday

### States can be used repeatedly

This allows us to use the same state machine in multiple places.

	LIST heatedWaterStates = cold, boiling, recently_boiled
	VAR kettleState = cold
	VAR potState = cold

	*	{kettleState == cold} [Turn on kettle]
		The kettle begins to boil and bubble.
		~ kettleState = boiling
	*	{potState == cold} [Light stove]
	 	The water in the pot begins to boil and bubble.
	 	~ potState = boiling

But what if we add a microwave as well? We might want start generalising our functionality a bit:

	LIST heatedWaterStates = cold, boiling, recently_boiled
	VAR kettleState = cold
	VAR potState = cold
	VAR microwaveState = cold

	=== function boilSomething(ref thingToBoil, nameOfThing)
		The {nameOfThing} begins to heat up.
		~ thingToBoil = boiling

	=== do_cooking
	*	{kettleState == cold} [Turn on kettle]
		{boilSomething(kettleState, "kettle")}
	*	{potState == cold} [Light stove]
		{boilSomething(potState, "pot")}		*	{microwaveState == cold} [Turn on microwave]
		{boilSomething(microwaveState, "microwave")}

or even...

	LIST heatedWaterStates = cold, boiling, recently_boiled
	VAR kettleState = cold
	VAR potState = cold
	VAR microwaveState = cold

	=== cook_with(nameOfThing, ref thingToBoil)
	+ 	{thingToBoil == cold} [Turn on {nameOfThing}]
	  	The {nameOfThing} begins to heat up.
		~ thingToBoil = boiling
		-> do_cooking.done

	=== do_cooking
	<- cook_with("kettle", kettleState)
	<- cook_with("pot", potState)
	<- cook_with("microwave", microwaveState)
	- (done)

Note that the "heatedWaterStates" list is still available as well, and can still be tested, and take a value.

#### List values can share names

Reusing lists brings with it ambiguity. If we have:

	LIST colours = red, green, blue, purple
	LIST moods = mad, happy, blue

	VAR status = blue

... how can the compiler know which blue you meant?

We resolve these using a `.` syntax similar to that used for knots and stitches.

	VAR status = colours.blue

...and the compiler will issue an error until you specify.

Note the "family name" of the state, and the variable containing a state, are totally separate. So

	{ statesOfGrace == statesOfGrace.fallen:
		// is the current state "fallen"
	}

... is correct.


#### Advanced: a LIST is actually a variable

One surprising feature is the statement

	LIST statesOfGrace = ambiguous, saintly, fallen

actually does two things simultaneously: it creates three values, `ambiguous`, `saintly` and `fallen`, and gives them the name-parent `statesOfGrace` if needed; and it creates a variable called `statesOfGrace`.

And that variable can be used like a normal variable. So the following is valid, if horribly confusing and a bad idea:

	LIST statesOfGrace = ambiguous, saintly, fallen

	~ statesOfGrace = 3.1415 // set the variable to a number not a list value

...and it wouldn't preclude the following from being fine:

	~ temp anotherStateOfGrace = statesOfGrace.saintly




## 3) List Values

When a list is defined, the values are listed in an order, and that order is considered to be significant. In fact, we can treat these values as if they *were* numbers. (That is to say, they are enums.)

	LIST volumeLevel = off, quiet, medium, loud, deafening
	VAR lecturersVolume = quiet
	VAR murmurersVolume = quiet

	{ lecturersVolume < deafening:
		~ lecturersVolume++

		{ lecturersVolume > murmurersVolume:
			~ murmurersVolume++
			The murmuring gets louder.
		}
	}

The values themselves can be printed using the usual `{...}` syntax, but this will print their name.

	The lecturer's voice becomes {lecturersVolume}.

### Converting values to numbers

The numerical value, if needed, can be got explicitly using the LIST_VALUE function. Note the first value in a list has the value 1, and not the value 0.

	The lecturer has {LIST_VALUE(deafening) - LIST_VALUE(lecturersVolume)} notches still available to him.

### Converting numbers to values

You can go the other way by using the list's name as a function:

	LIST Numbers = one, two, three
	VAR score = one
	~ score = Numbers(2) // score will be "two"

### Advanced: defining your own numerical values

By default, the values in a list start at 1 and go up by one each time, but you can specify your own values if you need to.

	LIST primeNumbers = two = 2, three = 3, five = 5

If you specify a value, but not the next value, ink will assume an increment of 1. So the following is the same:

	LIST primeNumbers = two = 2, three, five = 5


## 4) Multivalued Lists

The following examples have all included one deliberate untruth, which we'll now remove. Lists - and variables containing list values - do not have to contain only one value.

### Lists are boolean sets

A list variable is not a variable containing a number. Rather, a list is like the in/out nameboard in an accommodation block. It contains a list of names, each of which has a room-number associated with it, and a slider to say "in" or "out".

Maybe no one is in:

	LIST DoctorsInSurgery = Adams, Bernard, Cartwright, Denver, Eamonn

Maybe everyone is:

	LIST DoctorsInSurgery = (Adams), (Bernard), (Cartwright), (Denver), (Eamonn)

Or maybe some are and some aren't:

	LIST DoctorsInSurgery = (Adams), Bernard, (Cartwright), Denver, Eamonn

Names in brackets are included in the initial state of the list.

Note that if you're defining your own values, you can place the brackets around the whole term or just the name:

	LIST primeNumbers = (two = 2), (three) = 3, (five = 5)

#### Assiging multiple values

We can assign all the values of the list at once as follows:

	~ DoctorsInSurgery = (Adams, Bernard)
	~ DoctorsInSurgery = (Adams, Bernard, Eamonn)

We can assign the empty list to clear a list out:

	~ DoctorsInSurgery = ()


#### Adding and removing entries

List entries can be added and removed, singly or collectively.

	~ DoctorsInSurgery = doctorsSurgery + Adams 	~ DoctorsInSurgery += Adams  // this is the same as the above
	~ DoctorsInSurgery -= Eamonn
	~ DoctorsInSurgery += (Eamonn, Denver)
	~ DoctorsInSurgery -= (Adams, Eamonn, Denver)

Trying to add an entry that's already in the list does nothing. Trying to remove an entry that's not there also does nothing. Neither produces an error, and a list can never contain duplicate entries.


### Basic Queries

We have a few basic ways of getting information about what's in a list:

	LIST DoctorsInSurgery = (Adams), Bernard, (Cartwright), Denver, Eamonn

	{LIST_COUNT(DoctorsInSurgery)} 	//  "2"
	{LIST_MIN(DoctorsInSurgery)} 		//  "Adams"
	{LIST_MAX(DoctorsInSurgery)} 		//  "Cartwright"

#### Testing for emptiness

Like most values in ink, a list can be tested "as it is", and will return true, unless it's empty.

	{ DoctorsInSurgery: The surgery is open today. | Everyone has gone home. }

#### Testing for exact equality

Testing multi-valued lists is slightly more complex than single-valued ones. Equality (`==`) now means 'set equality' - that is, all entries are identical.

So one might say:

	{ DoctorsInSurgery == (Adams, Bernard):
		Dr Adams and Dr Bernard are having a loud argument in one corner.
	}

If Dr Eamonn is in as well, the two won't argue, as the lists being compared won't be equal - DoctorsInSurgery will have an Eamonn that the list (Adams, Bernard) doesn't have.

Not equals works as expected:

	{ DoctorsInSurgery != (Adams, Bernard):
		At least Adams and Bernard aren't arguing.
	}

#### Testing for containment

What if we just want to simply ask if Adams and Bernard are present? For that we use a new operator, `has`, otherwise known as `?`.

	{ DoctorsInSurgery ? (Adams, Bernard):
		Dr Adams and Dr Bernard are having a hushed argument in one corner.
	}

And `?` can apply to single values too:

	{ DoctorsInSurgery has Eamonn:
		Dr Eamonn is polishing his glasses.
	}

We can also negate it, with `hasnt` or `!?` (not `?`). Note this starts to get a little complicated as

	DoctorsInSurgery !? (Adams, Bernard)

does not mean neither Adams nor Bernard is present, only that they are not *both* present (and arguing).


#### Example: basic knowledge tracking

The simplest use of a multi-valued list is for tracking "game flags" tidily.

	LIST Facts = (Fogg_is_fairly_odd), 	first_name_phileas, (Fogg_is_English)

	{Facts ? Fogg_is_fairly_odd:I smiled politely.|I frowned. Was he a lunatic?}
	'{Facts ? first_name_phileas:Phileas|Monsieur}, really!' I cried.

In particular, it allows us to test for multiple game flags in a single line.

	{ Facts ? (Fogg_is_English, Fogg_is_fairly_odd):
		<> 'I know Englishmen are strange, but this is *incredible*!'
	}


#### Example: a doctor's surgery

We're overdue a fuller example, so here's one.

	LIST DoctorsInSurgery = (Adams), Bernard, Cartwright, (Denver), Eamonn

	-> waiting_room

	=== function whos_in_today()
		In the surgery today are {DoctorsInSurgery}.

	=== function doctorEnters(who)
		{ DoctorsInSurgery !? who:
			~ DoctorsInSurgery += who
			Dr {who} arrives in a fluster.
		}

	=== function doctorLeaves(who)
		{ DoctorsInSurgery ? who:
			~ DoctorsInSurgery -= who
			Dr {who} leaves for lunch.
		}

	=== waiting_room
		{whos_in_today()}
		*	[Time passes...]
			{doctorLeaves(Adams)} {doctorEnters(Cartwright)} {doctorEnters(Eamonn)}
			{whos_in_today()}

This produces:

	In the surgery today are Adams, Denver.

	> Time passes...

	Dr Adams leaves for lunch. Dr Cartwright arrives in a fluster. Dr Eamonn arrives in a fluster.

	In the surgery today are Cartwright, Denver, Eamonn.

#### Advanced: nicer list printing

The basic list print is not especially attractive for use in-game. The following is better:

	=== function listWithCommas(list, if_empty)
	    {LIST_COUNT(list):
	    - 2:
	        	{LIST_MIN(list)} and {listWithCommas(list - LIST_MIN(list))}
	    - 1:
	        	{list}
	    - 0:
				{if_empty}
	    - else:
	      		{LIST_MIN(list)}, {listWithCommas(list - LIST_MIN(list))}
	    }

	LIST favouriteDinosaurs = (stegosaurs), brachiosaur, (anklyosaurus), (pleiosaur)

	My favourite dinosaurs are {listWithCommas(favouriteDinosaurs, "all extinct")}.

It's probably also useful to have an is/are function to hand:

	=== function isAre(list)
		{LIST_COUNT(list) == 1:is|are}

	My favourite dinosaurs {isAre(favouriteDinosaurs)} {listWithCommas(favouriteDinosaurs, "all extinct")}.

And to be pendantic:

	My favourite dinosaur{LIST_COUNT(favouriteDinosaurs != 1:s} {isAre(favouriteDinosaurs)} {listWithCommas(favouriteDinosaurs, "all extinct")}.


#### Lists don't need to have multiple entries

Lists don't *have* to contain multiple values. If you want to use a list as a state-machine, the examples above will all work - set values using `=`, `++` and `--`; test them using `==`, `<`, `<=`, `>` and `>=`. These will all work as expected.

### The "full" list

Note that `LIST_COUNT`, `LIST_MIN` and `LIST_MAX` are refering to who's in/out of the list, not the full set of *possible* doctors. We can access that using

	LIST_ALL(element of list)

or

	LIST_ALL(list containing elements of a list)

	{LIST_ALL(DoctorsInSurgery)} // Adams, Bernard, Cartwright, Denver, Eamonn
	{LIST_COUNT(LIST_ALL(DoctorsInSurgery))} // "5"
	{LIST_MIN(LIST_ALL(Eamonn))} 				// "Adams"

Note that printing a list using `{...}` produces a bare-bones representation of the list; the values as words, delimited by commas.

#### Advanced: "refreshing" a list's type

If you really need to, you can make an empty list that knows what type of list it is.

	LIST ValueList = first_value, second_value, third_value
	VAR myList = ()

	~ myList = ValueList()

You'll then be able to do:

	{ LIST_ALL(myList) }

#### Advanced: a portion of the "full" list

You can also retrieve just a "slice" of the full list, using the `LIST_RANGE` function.

	LIST_RANGE(list_name, min_value, max_value)

### Example: Tower of Hanoi

To demonstrate a few of these ideas, here's a functional Tower of Hanoi example, written so no one else has to write it.


	LIST Discs = one, two, three, four, five, six, seven
	VAR post1 = ()
	VAR post2 = ()
	VAR post3 = ()

	~ post1 = LIST_ALL(Discs)

	-> gameloop

	=== function can_move(from_list, to_list) ===
	    {
	    -   LIST_COUNT(from_list) == 0:
	        // no discs to move
	        ~ return false
	    -   LIST_COUNT(to_list) > 0 && LIST_MIN(from_list) > LIST_MIN(to_list):
	        // the moving disc is bigger than the smallest of the discs on the new tower
	        ~ return false
	    -   else:
	    	 // nothing stands in your way!
	        ~ return true

	    }

	=== function move_ring( ref from, ref to ) ===
	    ~ temp whichRingToMove = LIST_MIN(from)
	    ~ from -= whichRingToMove
	    ~ to += whichRingToMove

	== function getListForTower(towerNum)
	    { towerNum:
	        - 1:    ~ return post1
	        - 2:    ~ return post2
	        - 3:    ~ return post3
	    }

	=== function name(postNum)
	    the {postToPlace(postNum)} temple

	=== function Name(postNum)
	    The {postToPlace(postNum)} temple

	=== function postToPlace(postNum)
	    { postNum:
	        - 1: first
	        - 2: second
	        - 3: third
	    }

	=== function describe_pillar(listNum) ==
	    ~ temp list = getListForTower(listNum)
	    {
	    - LIST_COUNT(list) == 0:
	        {Name(listNum)} is empty.
	    - LIST_COUNT(list) == 1:
	        The {list} ring lies on {name(listNum)}.
	    - else:
	        On {name(listNum)}, are the discs numbered {list}.
	    }


	=== gameloop
	    Staring down from the heavens you see your followers finishing construction of the last of the great temples, ready to begin the work.
	- (top)
	    +  (describe) {true || TURNS_SINCE(-> describe) >= 2 || !describe} [ Regard the temples]
	        You regard each of the temples in turn. On each is stacked the rings of stone. {describe_pillar(1)} {describe_pillar(2)} {describe_pillar(3)}
	    <- move_post(1, 2, post1, post2)
	    <- move_post(2, 1, post2, post1)
	    <- move_post(1, 3, post1, post3)
	    <- move_post(3, 1, post3, post1)
	    <- move_post(3, 2, post3, post2)
	    <- move_post(2, 3, post2, post3)
	    -> DONE

	= move_post(from_post_num, to_post_num, ref from_post_list, ref to_post_list)
	    +   { can_move(from_post_list, to_post_list) }
	        [ Move a ring from {name(from_post_num)} to {name(to_post_num)} ]
	        { move_ring(from_post_list, to_post_list) }
	        { stopping:
	        -   The priests far below construct a great harness, and after many years of work, the great stone ring is lifted up into the air, and swung over to the next of the temples.
	            The ropes are slashed, and in the blink of an eye it falls once more.
	        -   Your next decree is met with a great feast and many sacrifices. After the funeary smoke has cleared, work to shift the great stone ring begins in earnest. A generation grows and falls, and the ring falls into its ordained place.
	        -   {cycle:
	            - Years pass as the ring is slowly moved.
	            - The priests below fight a war over what colour robes to wear, but while they fall and die, the work is still completed.
	            }
	        }
	    -> top
