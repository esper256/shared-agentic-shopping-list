# Shared Agentic Shopping List

A shared household shopping list designed for **AI-first interaction** instead of checkboxes and manual list maintenance.

The idea is simple: your household keeps one shared shopping database, probably a Google Sheet, and gives your preferred AI assistants access to it. Then everyone can update the household state naturally in their own chatbot conversations.

Instead of opening a shopping app and finding the right item, you can just say things like:

> “We’re out of Rice Krispies.”
>
> “I opened the last toothpaste.”
>
> “We usually buy ketchup at FoodMaxx.”
>
> “I bought two gallons of milk.”

The AI records what you actually know without pretending the household is a perfectly tracked warehouse. “I used up a bottle of ketchup” might mean you should check whether another bottle is still in the pantry—not automatically add ketchup to the shopping list.

Later, when you say:

> “I’m heading to FoodMaxx. What do I need to know?”

…the agent can combine the shared household state into something more useful than a conventional list: what you definitely need, what you are probably getting low on, what is worth checking before you leave, and what you should skip because you already have plenty.

Multiple household members can use different AI assistants as long as those agents can access the same shared database. The database is the durable shared memory; individual chatbot conversations are not.

## Where this is going

The longer-term goal is to combine household inventory knowledge with retailer pricing and sales. Then an agent could say not only:

> “You’re almost out of dishwasher detergent.”

but also:

> “You still have some detergent, but the kind you normally buy is unusually cheap at Costco this week, so this may be a good time to stock up.”

That makes the system less of a shopping list and more of a **shared household purchasing assistant**.

## Project status

This project is currently defining the behavior and shared-data conventions that let independent AI agents cooperate safely. The first implementation will use Google Sheets for durable, human-readable shared storage.

The goal is to keep the human experience extremely simple: **talk to your AI normally, and let the agents maintain the list together.**
