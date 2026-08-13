---
title: Eat Agent
date: 2025-09-10
---

## Key Takeaways

### Core menu hierarchy

The required structure is:

**Menu Group → Section → Item**

Example:

**Food → Soups and Salads → Caesar Salad**

An item will not appear on the point of sale unless:

* The item is assigned to a section.
* The section is assigned to a menu group.
* The item is enabled for the correct location.
* The correct channel, such as in-store or online, is enabled.

A name and price alone are not enough.

### Menu groups and sections

Menu configuration depends on how the merchant organizes the restaurant.

A simple merchant may use:

* Group: Food
* Sections: Food and Drinks

A more detailed merchant may use:

* Groups: Breakfast, Lunch, Dinner, Happy Hour
* Sections: Burgers, Salads, Soups, Drinks, Appetizers

The agent should not assume one standard structure. It should inspect or ask how the merchant wants customers and staff to navigate the menu.

### Multi-location behavior

For multi-location accounts:

* The account can have a shared menu.
* Shared items should generally be managed from the **Account Menu**.
* An item may have different prices by location.
* Availability may differ by location.
* Online and in-store availability may be configured separately.
* A location-specific item can be created, but excessive location-level editing may make menu management confusing.

The recommended default is to manage common items at the account level and apply location-specific overrides where needed.

---

## Menu Agent Responsibilities

The Menu Agent should:

1. Create and maintain menu groups, sections, and items.
2. Validate the full menu hierarchy before publishing.
3. Assign items to the correct sections.
4. Confirm that every section belongs to a group.
5. Configure prices by location.
6. Configure online and in-store availability.
7. Distinguish between shared and location-specific items.
8. Detect incomplete or hidden menu items.
9. Preview or verify how the admin configuration appears on the POS.
10. Report configuration issues in plain language.

---

## Recommended Agent Validation Rules

Before marking an item as ready, the agent should verify:

```text
Item name exists
AND price exists
AND section is assigned
AND section belongs to a group
AND at least one location is assigned
AND at least one sales channel is enabled
```

For multi-location accounts, it should also verify:

```text
Each selected location has a valid price
Availability matches the intended location
In-store and online settings are intentional
Shared items are managed at the account level
```

### Suggested error messages

* **Missing section:** “This item will not appear on the menu because it is not assigned to a section.”
* **Section missing group:** “This section is not attached to a menu group, so its items will not appear on the POS.”
* **No location:** “This item exists at the account level but is not available at any location.”
* **No sales channel:** “This item is not enabled for in-store or online ordering.”
* **Location-only item:** “This item is managed only at one location and may be difficult to maintain across the account.”
* **Missing location price:** “A price has not been configured for one or more enabled locations.”

---

## Recommended Agent Workflow

### 1. Understand the merchant structure

Determine:

* Single-location or multi-location account
* Desired menu groups
* Desired sections
* Locations where the item should be sold
* In-store, online, or both
* Shared or location-specific item
* Same price or different prices by location

### 2. Build or select the hierarchy

The agent should either:

* Select an existing group and section, or
* Create the required group and section first.

It should never create an unassigned item unless it is intentionally saved as a draft.

### 3. Configure the item

Capture:

* Item name
* Description
* Base price
* Location-specific prices
* Section
* Menu group
* Location assignment
* Online availability
* In-store availability
* Optional modifiers or ingredients

### 4. Validate before publishing

Run hierarchy, pricing, location, and channel checks.

### 5. Preview the result

Show a side-by-side summary:

| Admin configuration       | POS result                     |
| ------------------------- | ------------------------------ |
| Group: Food               | Food tab or menu               |
| Section: Soups and Salads | Soups and Salads category      |
| Item: Caesar Salad        | Visible item under the section |
| In-store enabled          | Appears on POS                 |
| Online disabled           | Does not appear online         |

### 6. Return a completion report

Example:

```text
Menu item created successfully.

Item: Caesar Salad
Group: Food
Section: Soups and Salads
Locations: Downtown, Airport
In-store: Enabled at both locations
Online: Enabled at Downtown only
Price:
- Downtown: $12.00
- Airport: $14.00

Validation: Passed
POS visibility: Expected to appear under Food > Soups and Salads
```

---

## Next Steps

1. Create a demo menu with at least one group, two sections, and several items.
2. Document which menu properties are account-level versus location-level.
3. Define the Menu Agent’s required inputs and default assumptions.
4. Implement hierarchy and visibility validation rules.
5. Add support for location-specific prices and channel availability.
6. Produce a POS preview or a clear “where this item will appear” summary.
7. Add a diagnostic mode that identifies why an existing item is not visible.
8. Test these scenarios:

   * Item with no section
   * Section with no group
   * Item enabled at only one location
   * Different prices by location
   * Online-only item
   * In-store-only item
   * Shared item edited from the account level
   * Location-specific item created from a single location

## Suggested Menu Agent Objective

> The Menu Agent manages restaurant menu configuration across groups, sections, items, locations, prices, and sales channels. It ensures that every item is correctly structured and visible on the intended POS or online menu, while preventing incomplete, hidden, duplicated, or incorrectly assigned menu records.

https://schaeferscanalhouse.hrpos.heartland.us/menu

## Follow-Up Questions

### Menu Structure

1. Is the correct order **Menu Group → Section → Item**?
2. Does every item need to be assigned to a section before it appears on the point-of-sale menu?
3. Does every section also need to be assigned to a menu group?
4. Can one section belong to more than one menu group?
5. Can one item appear in multiple sections?
6. What happens if an item has a price but no section?
7. Is there a warning in the admin portal when an item or section is not properly assigned?

### Groups and Sections

8. What are some recommended ways to organize groups and sections?
9. Should groups represent meal periods, such as Breakfast, Lunch, and Dinner?
10. Should sections represent categories, such as Burgers, Soups, Salads, and Drinks?
11. Is it acceptable for a restaurant to have only one group, such as “Food”?
12. How does the group and section structure appear on the actual point-of-sale screen?
13. Can we see a side-by-side example of the admin setup and the customer-facing or point-of-sale menu?

### Multiple Locations

14. For a multi-location account, is the main menu shared across all locations?
15. Are menu groups and sections shared across locations, or can each location organize them differently?
16. Can the same item have a different price at each location?
17. Can an item be available at one location but unavailable at another?
18. Can availability be set separately for in-store and online ordering?
19. What is the best way to create an item that is only offered at one location?
20. Should all shared menu changes be made from the **Account Menu** rather than from an individual location?
21. If someone creates an item from an individual location, will it automatically appear at the account level?
22. How can we quickly identify which locations currently offer an item?

### Editing and Maintenance

23. Who should have permission to create or update menu groups, sections, and items?
24. Is there a way to preview the menu before publishing changes?
25. Do menu changes take effect immediately, or is there a publishing or syncing step?
26. Can we temporarily hide an item without deleting it?
27. What is the safest process for removing an item that is no longer sold?
28. Is there an audit history showing who changed an item, price, section, or location assignment?
29. Are there reports that identify items that are missing a section, group, price, or location?
30. What are the most common setup mistakes new users make?

## Recommendations

### 1. Teach the Menu Hierarchy First

Use one simple rule throughout the training:

**Group → Section → Item**

Example:

**Lunch → Burgers → Cheeseburger**

Explain that an item will not appear on the point-of-sale menu unless:

* The item is assigned to a section.
* The section is assigned to a group.
* The item is enabled for the correct location and ordering channel.

### 2. Provide a Side-by-Side Demonstration

Show the admin portal on one side and the resulting point-of-sale menu on the other. During the demonstration:

* Create a menu group.
* Create a section and attach it to the group.
* Create an item and attach it to the section.
* Add the price.
* Enable it for a location.
* Show where it appears on the point of sale.

Also demonstrate what happens when the section or group assignment is missing.

### 3. Use a Simple Training Example

Build a small demo menu instead of using a large live menu.

Example:

* **Group:** Food
* **Sections:** Burgers, Soups and Salads, Drinks
* **Items:** Cheeseburger, Tomato Soup, Caesar Salad, Soda

For a multi-location example, use two locations and give one item different prices at each location.

### 4. Manage Shared Items from the Account Menu

For multi-location merchants, make the **Account Menu** the main place for creating and editing shared items. This reduces duplicate items and makes location-specific pricing and availability easier to manage.

Only create an item directly within a location when it is truly unique to that location.

### 5. Establish a Standard Naming Structure

Use consistent names for groups, sections, and items. Avoid vague names such as “Food 1,” “New Section,” or “Test Item.”

Recommended examples:

* Groups: Breakfast, Lunch, Dinner, Happy Hour
* Sections: Burgers, Appetizers, Salads, Beverages
* Items: Classic Cheeseburger, Caesar Salad, Iced Tea

### 6. Create a Pre-Publish Checklist

Before considering an item complete, confirm:

* The item has a clear name.
* The item has a price.
* The item is assigned to a section.
* The section is assigned to a group.
* The correct locations are selected.
* In-store and online availability are correctly set.
* Any location-specific prices are entered.
* The item has been previewed or tested on the point of sale.

### 7. Clarify Shared Versus Location-Specific Settings

Provide a simple reference chart explaining which settings are:

* Shared across the account.
* Configurable by location.
* Configurable by ordering channel.
* Best edited from the Account Menu.
* Best edited from the individual location.

### 8. Use a Dedicated Demo Account

Prepare a multi-location demo account before future training sessions. The demo should already contain:

* At least two locations.
* Multiple groups and sections.
* Shared items.
* One location-only item.
* Different prices by location.
* Different in-store and online availability settings.

### 9. Document Common Problems

Create a troubleshooting guide covering issues such as:

* Item does not appear on the point of sale.
* Item has no section.
* Section has no group.
* Item is not enabled for the location.
* Item is enabled online but not in-store.
* Incorrect location price is displaying.
* Duplicate items were created at different levels.

### 10. Confirm Understanding with a Practice Exercise

Ask each team member to create one sample item and verify that it appears correctly on the menu. This will help confirm that they understand the relationship between groups, sections, items, locations, prices, and availability.


## Menu Search and Reporting

1. **Use the menu hierarchy as the source of truth:**
   **Menu Group → Section → Menu Item**

2. **Resolve the user’s term before querying.** Determine whether “beer,” “drinks,” or another term refers to:

   * A menu group
   * A section
   * A specific menu item

3. **Aggregate all child items for category questions.**
   For “How many beers did I serve yesterday?”, if Beer is a section, total the sales of every menu item assigned to that section.

4. **Use account ID and location ID first.** Then filter by group, section, or item to reduce irrelevant matches.

5. **Do not rely only on semantic search.** Use semantic search to interpret the request, then use structured menu IDs and hierarchy relationships for the final query.

6. **Handle inconsistent merchant naming.** The same concept may appear in multiple sections or groups, such as Drinks under Breakfast and Dinner. Matching should use IDs and parent relationships, not names alone.

7. **Request clear data from ATL.** The required dataset or API should return:

   * Account and location IDs
   * Group ID and name
   * Section ID and name
   * Item ID and name
   * Parent-child relationships
   * Availability by location
   * Effective dates or active status

8. **Build a category-resolution service.** It should return the matched entity type, confidence, parent hierarchy, and all relevant item IDs.

## Immediate Next Steps

* Document the exact menu hierarchy and data fields required from ATL.
* Follow up on the existing ATL email and schedule a working session if no response is received.
* Test category aggregation using Beer, Drinks, Breakfast, and one specific item.
* Define how ambiguity will be handled when the same section name exists under multiple groups.
* Separate account memory and user-preference memory discussions into distinct sessions.

## API and Penetration Testing

* Prioritize streaming changes because they are more likely to require penetration testing.
* Confirm with security and Apigee whether new memory APIs can be covered under the existing API pattern or require separate testing.
* Reserve a stable testing environment early; penetration testing may require two to three weeks plus queue time.
* Avoid freezing the main development environment. Prefer a dedicated test instance where practical.
* Consider status updates instead of full streaming only if that still meets the product experience and reduces implementation or security risk.
* Reassess the October timeline after confirming the penetration-testing scope and environment plan.
