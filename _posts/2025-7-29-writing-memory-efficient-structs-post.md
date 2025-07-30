---
layout: post
title: "Writing memory efficient C structs"
date: 2025-07-29
tags: [blog, tutorial, C]
---

A struct in C is the best way to organize your data so that you can easily use the data later in your program. However, there are a few caveats to C structures, mainly how their memory works.
Note that I'm making a lot of assumptions about type sizes and standards. These can differ between systems and ABI's, so for example, the alignment value of an int isn't strictly 4, meaning some sizes differ from mine.

## Our struct 
```c
struct Monster {
	bool is_alive; // Used to see whether or not the monster is alive
	int health; // Health of the monster
	int damage_hit; // Damage they deal per hit
	char name[64]; // Name of the monster with a max of 63 characters and one \0, 64 bytes allocated to the stack
	float x_position; // The x_position of the monster in the game
	float y_position; // The y_position of the monster in the game
	bool can_fly; // Boolean whether or not monster can fly
	bool can_swim; // Boolean whether or not monster can swim
	int speed; // Speed of monster whilst walking
	bool is_poisoned; // If the monster is poisoned
	bool has_armor; // If the monster has any protection
};
```

Here I've defined a basic Monster struct. It has a lot of basic fields which hold some data about this monster.
Before trying to reduce the size of this struct we should probably look at how large this struct is at the moment. So let's count all of the bytes!
```c
struct Monster {
	bool is_alive; // Boolean is 1 byte
	int health; // Int is 4 bytes
	int damage_hit; // 4 bytes
	char name[64]; // 64 times char, which is 1 byte each; 64 bytes total
	float x_position; // Float is 4 bytes
	float y_position; // 4 bytes
	bool can_fly; // 1 byte
	bool can_swim; // 1 byte
	int speed; // 4 bytes
	bool is_poisoned; // 1 byte
	bool has_armor; // 1 byte
};
```
So let's see, we have 5 Booleans, 3 integers, 2 floats and one 64 byte string... That should be 89 bytes! Let's test this theory:
```c
sizeof(struct Monster); // => 96
```

The actual size of the struct is 96 bytes, not the 89 bytes we initially predicted. Let's look at our struct in memory:

| Offset | Size (bytes) | Member      |
| ------ | ------------ | ----------- |
| 0      | 1            | is_alive    |
| 1-3    | 3            | **Padding** |
| 4-7    | 4            | health      |
| 8-11   | 4            | damage_hit  |
| 12-75  | 64           | name        |
| 76-79  | 4            | x_position  |
| 80-83  | 4            | y_position  |
| 84     | 1            | can_fly     |
| 85     | 1            | can_swim    |
| 86-87  | 2            | **padding** |
| 88-91  | 4            | speed       |
| 92     | 1            | is_poisoned |
| 93     | 1            | has_armor   |
| 94-95  | 2            | **Padding** |

So now that number makes a bit more sense. As you can see from the memory layout table above there is a total of 6 bytes of padding added to the struct, which is exactly the memory we were missing.
But why is this padding even added? The reason for this padding is because each type has its own alignment requirements, meaning that the compiler must place the it at a memory address such that it's a multiple of its alignment value, so in this case each ```int``` field requires to be at an alignment of 4 (depends on system and ABI), other types have different alignment requirements, like for booleans and chars it's 1 on most systems and for double it's 8 (again, usually). This padding is added because misalligned access can be slower or even illegal on some hardware. Add offset 94 and 95 we also have some padding, this padding is called tail padding because it isn't there to allign any other field after it. Tail padding ensures the total struct size is a multiple of the struct’s alignment, which is the maximum alignment requirement of any member. This guarantees that if you create an array of structs, each element remains properly aligned.. So in this case one of the fields with the largest alignment value is the ```health``` field, which has an alignment value of 4. Now we take the size of the struct (excluding the tail padding), which is 94 and round it up to the nearest multiple of 4, so 96.

## Padding reduction

Can we reduce padding? Yes! As you can see at byte offset 84 to 85, there actually isn't any padding added between them. This is because can_fly and can_swim have the same alignment value (1 byte). Meaning that when the compiler sees two fields of the same size it combines them. We can use this to our advantage by grouping all of the fields together with the same alignment value. It's best to order your struct from largest field to smallest, this minimizes size because types with a larger alignment value will have to add padding way less because the fields before it take up all the memory:
```c
struct Monster {
	char name[64];
	int health;
	int damage_hit;
	int speed;
	float x_position;
	float y_position;
	bool can_fly;
	bool can_swim;
	bool is_alive;
	bool is_poisoned;
	bool has_armor;
};
```
So when running sizeof struct Monster now we would hopefully see a different result:
```c
sizeof(struct Monster); // => 92
```
Great! We've brought the size of our struct down by 4 bytes already. Let's look at how our struct looks in memory now:

| Offset | Size | Member      |
| ------ | ---- | ----------- |
| 0-63   | 64   | name        |
| 64-67  | 4    | health      |
| 68-71  | 4    | damage_hit  |
| 72-75  | 4    | speed       |
| 76-79  | 4    | x_position  |
| 80-83  | 4    | y_position  |
| 84     | 1    | can_fly     |
| 85     | 1    | can_swim    |
| 86     | 1    | is_alive    |
| 87     | 1    | is_poisoned |
| 88     | 1    | has_armor   |
| 89-91  | 3    | **Padding** |

As you can see, by only **reordering** our struct we've already made a reduction in the amount of memory our struct uses. Say we have a thousand monsters, we’ve reduced the memory usage of the monsters from 96KB to 92KB, but we can go further!


## Derived state
Say you have a Person struct and that struct holds a field about the person's age, and you want to find out if the person is a legal adult. In this cases, you can derive this state dynamically instead of storing it directly, which can reduce struct size. This principle can be used to cut down on the amount of boolean fields needed on a struct, but it requires some logical thinking. 
In our case we can deprecate the is_alive field, since we already have the health field, so as long as health > 0 the monster is alive.
```c
struct Monster {
	char name[64];
	int health;
	int damage_hit;
	int speed;
	float x_position;
	float y_position;
	bool can_fly;
	bool can_swim;
	// bool is_alive; Unnecessary, just check if health > 0
	bool is_poisoned;
	bool has_armor;
};
```
So now running sizeof struct Monster again we get:
```c
sizeof(struct Monster); // => 88
```
We brought the size of the struct down 4 bytes by just removing a singular boolean? Remember: The tail padding is just the size of the struct rounded up to a multiple of the maximum alignment value. So in this case we go from 89 bytes, which is rounded up to 92 bytes, to 88 bytes, which already is a multiple of the maximum alignment value.

## Types
Another way to cut down on the overall size of our struct is by using the smallest appropriate size available. Take for example the health field. The health of a monster is currently represented by a 4 byte signed integer, meaning that the health can range from -2^31 to 2^31-1 (2^31 ~= 2 million). We can already see that half of the integers potential is unused because the health of a monster should never be negative, so an unsigned int would fit way better. However, this still leaves us with over 4 million possible integer values, which is simply way too much to represent the health of a monster. This is where the stdint.h header comes in handy. This header defines a lot of useful int types which lets us use integers with a specific number of bits. The smallest of these being uint8_t (unsigned 8 bit int). An unsigned 8 bit int has a range from 0 to 255, which I reckon would be enough in most cases, but just to be sure we will use a uint16_t, which has a much larger range of over 65 thousand. The same can be done for damage_hit and the speed field can easily be represented in a uint8_t. The x_position and y_position can unfortunately not get a smaller type nor can a float be unsigned. This leaves us with the following struct:
```c
struct Monster {
	char name[64];
	float x_position;
	float y_position;	
	uint16_t health;
	uint16_t damage_hit;
	uint8_t speed;
	bool can_fly;
	bool can_swim;
	bool is_poisoned;
	bool has_armor;
};

sizeof(struct Monster); // => 84
```

## Bitfields
Some values can easily be represented in just 1 bit, take the booleans for example; it's just 0 or 1, however the program still requires the full byte to be used, meaning you waste a whole 7 bits per boolean. Here's where bitfields come into play. A bitfield is a data structure which maps to adjacent bits which have been allocated for a specific reason. In our case this allows us to represent all the booleans - which are currently represented in 4 bytes - in just 4 bits. The syntax looks as follows:
```c
struct Monster {
	char name[64];
	float x_position;
	float y_position;	
	uint16_t health;
	uint16_t damage_hit;
	uint8_t speed;
	// Note that bool bitfields ARE allowed following the C23 standard, altough for maximum portability we'll use unsigned instead
	unsigned can_fly : 1;
	unsigned can_swim : 1;
	unsigned is_poisoned : 1;
	unsigned has_armor : 1;
};

sizeof(struct Monster); // => 80
```

## Use enums for ID
Our last method on how to reduce the size of your struct and this can be the most impactful one. If you store any name used for identification as a string directly on the struct, let's say the model version of a phone or the name of a specific monster you've implemented in your game, you're doing it wrong. Using this implementation you'd be allocating a totally new string for each Monster you're creating, this is a total waste of memory. Instead it would be best to use enum in this case, which usually has a size of 4 bytes (not guaranteed). Just define an enum with all of the monster names or phone models you want to have:
```c
enum MonsterName {
	GIANT,
	ZOMBIE,
	SKELETON,
	SPIDER,
	GOBLIN,
	// etc...
};
```
Then just use the enum instead of a whole new string:
```c
struct Monster {
	enum MonsterName name;
	float x_position;
	float y_position;	
	uint16_t health;
	uint16_t damage_hit;
	uint8_t speed;
	unsigned can_fly : 1;
	unsigned can_swim : 1;
	unsigned is_poisoned : 1;
	unsigned has_armor : 1;
};

sizeof(struct Monster); // => 20
```
20 bytes. We have brought the size of your struct down from 96 bytes to a measly 20 bytes. Nearly a 5-fold reduction in memory. For a thousand monster you'd go from 96Kb to just 20Kb of memory, shaving off a total of 76Kb of memory.
There are still some potential improvements to be made, like you can derive a lot of state from just the name of the Monster, but I'm leaving that up as a challenge for the reader.

## Trade-offs
Does this mean that you should immediately hyper optimize every struct you write from now on? As most things it depends on the context. Writing structs which are memory efficient is especially important in performance sensitive systems or programs with limited memory. However say you just have one or two structs in your program it could be worth the extra bytes for readability purposes. 
Also, some of these methods can lead to unexpected behavior if not handled properly, like integer overflows if you use an integer type which is too small.

## Note
I wrote this blog as a teen, typing it up on my old school computer because I recently stumbled upon data oriented development and thought it was interesting and wanted to write a blog about it. I'm by far an expert and I could be wrong in some specifications. If you find any errors feel free to open an issue on GitHub!
