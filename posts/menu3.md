---
title: "Ingredient Context and Modifiers"
date: 2024-08-12
---

## Summary

The discussion clarified how **ingredient context, modifiers, and miscellaneous/open items** work together in the POS configuration.

The **context** feature lets one ingredient support different ordering situations without creating separate versions of the same ingredient. An ingredient has a base price, and each context applies a multiplier to that price. For example, if cheese costs $1, a context of **Extra = 2x** would charge $2, **On Side = 1x** would charge $1, and **Free/No Charge = 0x** would charge $0. This avoids creating separate ingredients such as “Cheese,” “Extra Cheese,” and “Cheese on Side.”

The context multiplier applies to the ingredient price that is relevant in that configuration. For example, if lettuce is configured at **$0.50 within the Burger Toppings modifier**, and its Extra context is 2x, selecting extra lettuce at the POS would result in a **$1.00 charge**.

A **modifier** is a group of ingredients or choices associated with an item. For example, a Burger Toppings modifier could contain lettuce and tomato, while a Fries modifier could contain chili and cheese. Modifier configuration can also control things such as how many choices the customer can select and how many selected ingredients are included for free.

The **Number of Included Ingredients** setting allows a certain number of selections from a modifier to be included without an additional charge, even when those ingredients normally have prices.

The discussion also clarified the concept of **open/miscellaneous ingredients and items**. Checking the **Miscellaneous Ingredient** box turns an ingredient into an open ingredient. Instead of always using a predefined price, the POS prompts the employee to enter the price when it is rung up. The same concept exists for items through the **Miscellaneous Item** checkbox. Therefore, an AI instruction such as “create an open item” or “create an open ingredient” should translate into creating the appropriate record and enabling its miscellaneous checkbox.

Other item-level settings briefly discussed included the **Non-Revenue flag**, typically used for things such as gift cards; an **Alcohol flag**, which can affect integrations such as delivery platforms; and measurement-based pricing options that may be used for products sold by weight or even time.

## Key configuration relationships

A useful mental model is:

**Ingredient → Context → Modifier → Item**

An **ingredient** defines the underlying option.
A **context** changes how its price behaves in situations such as Extra or On Side.
A **modifier** groups ingredients and establishes how those ingredients behave within a particular choice group.
An **item** such as a Burger Combo attaches the relevant modifiers and has its own item price and properties.

For AI-driven configuration, the important distinction is that requests such as **“extra lettuce” should generally modify the context/multiplier rather than create another lettuce ingredient**.

## Follow-up questions

1. When an ingredient has both a base ingredient price and a different price inside a modifier, which price should the context multiplier use in every scenario? The example suggests it uses the modifier-specific price, but this should be formally confirmed.

2. Are context multipliers limited to values such as 0x, 1x, and 2x, or can users configure arbitrary values such as 0.5x, 1.5x, or 3x?

3. Can the same ingredient have different context behavior depending on the modifier or item, or is its context configuration global everywhere the ingredient is used?

4. If an ingredient is already included by default on an item, how should **Extra** pricing work? For example, if standard cheese is included in the burger price, does Extra Cheese charge the full ingredient price × multiplier, or only the incremental portion?

5. How does **Number of Included Ingredients** interact with context? If one ingredient is included for free but the customer selects it as Extra, does the inclusion apply before or after the context multiplier?

6. Can contexts represent actions other than pricing, such as **Light, No, Substitute, Half, First Half, Second Half, or On Side**, and can each have separate multipliers?

7. For a 0x context, is the ingredient recorded on the ticket with a $0 price, or does the system treat it differently?

8. When an item or ingredient is marked **Miscellaneous**, are there permissions or price limits controlling which employees can enter an open price?

9. Should an open item normally have a configured default price of $0, a blank price, or does the miscellaneous checkbox override whatever price is stored?

10. Are modifier-level ingredient prices overrides of the ingredient's normal price, or are they additional charges layered on top of the ingredient price?

## Recommendations for the AI implementation

The AI should translate user language into **configuration intent**, rather than reproducing the wording literally. For example, “Make extra cheese cost twice as much” should update the Cheese ingredient's Extra context to **2x**, rather than creating a new “Extra Cheese” ingredient.

It would be useful to teach the AI a small set of explicit mappings:

* **“Open item” → Create/update Item + enable Miscellaneous Item**
* **“Open ingredient” → Create/update Ingredient + enable Miscellaneous Ingredient**
* **“Extra X costs double” → Add/update Extra context for X = 2x**
* **“X on the side costs the same” → On Side context = 1x**
* **“Don't charge for X in this context” → Context multiplier = 0x**
* **“First two toppings are free” → Modifier Number of Included Ingredients = 2**
* **“Add lettuce and tomato choices to the burger” → Create/use modifier and associate those ingredients**

I would also recommend preventing the AI from creating duplicate ingredients unless the user explicitly needs separate inventory or reporting records. Before creating “Extra Lettuce,” “Side Lettuce,” or similar records, it should check whether an existing **Lettuce ingredient + context** can satisfy the request.

Finally, the product documentation or AI schema should make the pricing hierarchy explicit. The biggest potential source of confusion is determining **which ingredient price is multiplied when an ingredient appears in different modifiers or items**. Defining that rule clearly will make AI-generated configurations much more reliable.

## Summary

This section clarified three important behaviors: **default ingredients**, **context pricing on defaults**, and **sub-items/nested modifiers**.

When an ingredient is marked as **default** on an item, the POS assumes that ingredient already comes with the item. For example, if lettuce is defaulted on a burger, tapping the selected lettuce can effectively mean **“No Lettuce”** rather than adding another lettuce. This avoids needing a separate “No” context or separate “No Lettuce” ingredient.

Ingredients that have context options are visually indicated by the **three-circle icon**. On the POS, staff can long-press the ingredient to access its available contexts, such as Extra or other configured behaviors.

A key pricing clarification is that **context pricing accounts for ingredients that are already included**. For example, if lettuce is included on the burger at a $0.50 value and Extra is configured as 2x, choosing Extra Lettuce does **not necessarily add the full $1.00 on top of the item**. Because one portion is already included, the system charges only for the additional portion—$0.50 in this example. By contrast, if tomato is not already included and its base modifier price is $0.50, Extra Tomato at 2x can result in a $1.00 charge.

The session then introduced **sub-items**. A sub-item is used when an ingredient within a modifier needs to have its **own modifiers or further customization**.

For example, the Burger Combo has a side modifier with:

* Fries
* Green beans

Fries are the default side. However, fries themselves may need additional options such as:

* Add chili
* Add cheese

Instead of creating a second special version of fries just for the Burger Combo, the **Fries ingredient is linked to the existing Fries item as its sub-item**. The Fries item already contains the chili and cheese modifiers. When the Fries ingredient is selected inside the Burger Combo, the POS looks to the linked Fries item and exposes those same settings.

This creates a nested structure such as:

**Burger Combo → Burger Sides modifier → Fries ingredient → Fries sub-item → Chili/Cheese modifier**

On the POS, a linked sub-item creates an additional tab or level where staff can modify that selected ingredient further. Without the sub-item link, fries would simply behave as an ingredient that could be selected or removed; staff would not be able to drill into the fries and add chili or cheese.

The same Fries item can also continue to exist as a **standalone menu item**. This means the restaurant can maintain one configuration for fries and reuse its modifier behavior both when fries are sold independently and when they appear as a side within another item.

Other strong examples of sub-items mentioned were:

* **Baked potato** → butter, sour cream, cheese, bacon, etc.
* **Side salad** → dressing selection and other salad modifications
* **Pizza configurations** → potentially multiple nested customization levels

Sub-items can apparently be nested multiple levels deep, although that can make configurations significantly more complex.

## Important rule discovered about context pricing

The earlier model of simply saying:

**Context charge = ingredient price × multiplier**

is incomplete when the ingredient is already included by default.

A better conceptual model is:

**Final additional charge depends on the context multiplier and how much of the ingredient is already included in the item.**

For example:

* Lettuce price = $0.50
* Lettuce is already included/defaulted
* Extra = 2x
* Total desired quantity/value = $1.00
* Already included value = $0.50
* **Incremental charge = $0.50**

Whereas:

* Tomato price = $0.50
* Tomato is not already included
* Extra = 2x
* **Charge = $1.00**

That distinction should be captured explicitly in any AI logic or documentation.

## Follow-up questions

1. **How exactly is the incremental context price calculated for default ingredients?** Is the formula always `base price × multiplier − already included amount`, or are there other pricing rules involved?

2. If a default ingredient has a **0x or 1x context**, what happens financially? For example, does “On Side” for an included ingredient remain free because the original portion is already included?

3. If an ingredient is defaulted but the modifier also has **Number of Included Ingredients**, which inclusion rule takes precedence?

4. Does turning off a default ingredient always automatically generate a kitchen instruction such as **“No Lettuce,”** or can that wording be customized?

5. Can a context itself represent **No**, or is the preferred setup always to use the default-selected state and deselect it?

6. Is the three-circle context indicator visible only when at least one context is configured, or does it also indicate other ingredient actions?

7. Can **every ingredient** be linked to a sub-item, or are there restrictions on which item types can act as sub-items?

8. Can multiple different ingredients point to the **same sub-item**? For example, could “Regular Fries” and “Large Fries” both inherit options from one Fries item?

9. When a sub-item item has its own base price, is that price ignored when it is being used as a sub-item, with only its modifier charges inherited?

10. Which properties of the linked item are inherited by the sub-item relationship? For example: modifiers, default ingredients, contexts, taxes, availability, kitchen routing, price, or all of the above?

11. If the linked Fries item is later updated—for example, a new “Loaded” modifier is added—does every parent item using Fries as a sub-item immediately inherit that update after publishing?

12. What happens if sub-items are nested several layers deep and one of the links creates a circular reference? Does the admin portal prevent that?

13. Is there a recommended maximum nesting depth for usability even though the system technically permits multiple levels?

14. When a sub-item is unavailable or out of stock, does that automatically affect the parent ingredient wherever it is used?

15. If fries are defaulted as the side and the customer switches to green beans, does the POS automatically remove the fries sub-item tab and replace it with any sub-item configuration attached to green beans?

## Recommendations

The AI configuration logic should treat **default state, context, and sub-item relationships as separate concepts**.

A request such as **“The burger comes with lettuce”** should default/select the Lettuce ingredient on the relevant modifier. A request such as **“Let customers order extra lettuce”** should configure an Extra context on the existing Lettuce ingredient rather than create “Extra Lettuce.” And a request such as **“If they choose fries, let them add chili and cheese”** should identify that fries need further customization and use a **sub-item relationship**, rather than duplicating the chili and cheese modifiers directly across every combo.

The AI should also recognize phrases that strongly imply a sub-item. Examples include:

* “When they choose this side, ask them…”
* “Let them customize the fries further.”
* “If they pick salad, then ask for dressing.”
* “This option has its own modifiers.”
* “Use the same options as the standalone fries.”
* “I don't want to rebuild the modifiers for every combo.”

Those should map conceptually to:

**Ingredient → link Sub-Item → existing Item with desired modifiers**

A second recommendation is to add a **reuse-first rule**. Before creating new modifiers, ingredients, or duplicate items, the AI should check whether an existing standalone item already contains the needed configuration and could be referenced as a sub-item.

For example, instead of creating:
**Burger Fries → Burger Fries Chili → Burger Fries Cheese**

the preferred architecture is:

**Fries Item**
→ Chili / Cheese modifiers

and then:

**Burger Combo**
→ Side modifier
→ Fries ingredient
→ Sub-item = Fries Item

This creates one reusable source of truth.

Finally, documentation for the AI should distinguish between three different types of nesting:

**Item → Modifier → Ingredient** is normal item configuration.

**Ingredient → Context** changes how that ingredient behaves or is priced.

**Ingredient → Sub-Item → Modifier → Ingredient** means the ingredient itself can be customized further.

That distinction appears central to understanding how this menu model is intended to work.
