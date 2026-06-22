# TakeMeter — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Community and Labels
Community: r/leagueoflegends

Why: I personally enjoy playing this game with my friends and can understand the different frustrations or joys that players have with the updates or changes that get made. With this community, there is a massive variety to the posts that are created, with multiple categories. When players complain about a certain character, they could be arguing about the character itself, or the build that a certain character can go and how it scales. There are also discussions among the community about the international scene and the competitions between the regions. I previously had a conversation where someone I knew argued that a team lost a very important match simply because of an item bug. These different nuances to the game make it extremely interesting and providing proper labels can serve to be a difficult challenge.

### Label 1: balance_discussion
- Posts under this label focus on the game's mechanics, patch notes, champion kits, or itemization, arguing for or against changes based on gameplay data, mechanics, or win rates.
- Example: "The new item buffs made ADC items way too cheap. Completing Infinity Edge at 10 minutes makes the mid-game impossible to play for top laners because of the raw stats spike."
- Example: "Can we talk about Phreak’s reasoning for the patch 16.4 jungle changes? Lowering camp XP just forces junglers to gank 24/7 instead of farming, which completely ruins power-farming champions like Graves."
- Borderline: "I am so sick and tired of playing against Shaco. His clone is annoying, his boxes are annoying, and he ruins every single game he's in. Please delete this champ."
       - It mentions a champion kit, but it offers zero tactical analysis, stats, or structural arguments. It leans heavily into pure emotional venting.

### Label 2: esports_discourse
- These posts center entirely on the competitive professional scene (LCS, LCK, LEC, LPL, Worlds), including roster rumors, match analysis, player drama, or tournament formats.
- Example: "T1 vs Gen.G Post-Match Discussion: What an insane Baron steal by Oner in game 5 to secure the reverse sweep!"
- Example: "Rumor: G2 is looking to replace their top laner for the upcoming Summer Split according to Sheep Esports."
- Borderline: "Faker’s Azir build in game 3 proved that Nashor's Tooth is objectively bait on this patch. Here is why Liandry's does more damage per gold spent in teamfights."
       - It references a pro player in a pro match (esports_discourse), but the actual substance of the post is a deep dive into itemization mechanics and math (balance_discussion).

### Label 3: community_meta
- Posts under this label address peripheral aspects of League of Legends, such as client bugs, toxic player behavior, ranked climbing mentality, cosmetic skins, or the state of the subreddit itself.
- Example: "The Riot Client has crashed three times today during champ select, losing me 15 LP and giving me a queue penalty for dodging. Is this happening to anyone else?"
- Example: "The vanguard anti-cheat system is still blocking my discord overlay. This has been a known issue for months and it's getting ridiculous."
- Borderline: "Ranked is completely unplayable right now because every game has someone who rages and goes AFK after giving up first blood. Riot needs to fix matchmaking or punish people harder."
       - It mentions community toxicity (community_meta), but it also claims the game state/matchmaking is "unplayable," which borders on a structural game complaint.

### Boundary Rules
If a post complains about a champion or item but provides zero analytical reasoning, math, or mechanical breakdown, it will be classifed as `community_meta` (venting about game experience). If it critiques numbers or mechanics, it falls under `balance_discussion`. If a post uses a pro player or pro match merely as a springboard to argue about item math, champion design, or patch health, it would be classified as `balance_discussion`. If the main point is evaluating the pro player's performance or the match outcome, it is `esports_discourse`.

---

## Hard Edge Cases
Case: A post arguing that a professional player's item build or lane swap strategy is ruining solo-queue games.
Reason: It sits directly on the border of `esports_discourse` (discussing pro tactics), `balance_discussion` (discussing strategy/item efficiency), and `community_meta` (discussing how it ruins the community's ranked games).

Solution: I will implement a *primary intent rule* during annotation. I will ask: If you removed the professional scene reference, does the post still stand as a coherent argument about game balance? If yes, it is `balance_discussion`. If the core frustration is the trickle-down toxicity or teammate behavior in solo-queue, it is `community_meta`. If the post's primary engagement is evaluating the pro scene's impact, it is `esports_discourse`.

---

## Data Collection Plan
Source: Data will be scraped directly from r/leagueoflegends using the Reddit API (PRAW), targeting both the "Hot" and "Top" sections of the past month to ensure an organic variety of content.

Target Distribution: I will aim for an equal distribution across the three categories (~65–70 posts per label to cross the 200 threshold).

Handling Underrepresented Labels: If after collecting 200 random posts a specific label has fewer than 50 examples, I will use Reddit's native search feature using explicit keywords (e.g., searching "client crash", "Vanguard", "patch notes", or "rework") to targetedly oversample the missing class until a healthy balance is achieved.

---

## Evaluation Metrics
Because public community data can naturally form imbalanced distributions, relying solely on accuracy is dangerous. I will use the *Macro F1-Score* as my primary metric because it averages the F1-scores of each class equally, ensuring the model doesn't just score high by over-predicting a dominant category (e.g., esports_discourse during tournament seasons). *Precision* matters for balance_discussion so a tool filtering for balance threads doesn't get flooded with emotional rants. *Recall* matters for community_meta to ensure client bugs or system errors are captured completely without missing critical user feedback.

---

## Definition of Success
The Baseline: The fine-tuned model must decisively outperform the zero-shot Llama-3.3-70b baseline on the test dataset. For this classifier to be "good enough" for a real-world community analytics tool or automated moderator tagger, it must achieve a minimum Macro F1-Score of 0.82 and an overall Accuracy of 85% on the test set. Additionally, individual class precision must not drop below 0.80, guaranteeing that community users can trust the filtered content feeds.

---

## AI Tool Plan
1. Label Stress-Testing: Before starting my 200-example annotation phase, I will pass my exact label definitions and boundary rules to an LLM (e.g., Claude or ChatGPT). I will instruct the AI to act as a cynical `r/leagueoflegends` user trying to break the classifier, generating 10 highly ambiguous posts that sit directly on the knife's edge between balance_discussion and community_meta. If the AI produces examples that make me hesitate for more than a few seconds, I will write an explicit sub-rule into my Boundary Rules section to resolve that specific linguistic pattern before manual labeling begins.

2. Annotation Assistance: I will use an LLM via a custom script or interface to generate a "silver standard" pre-label for my raw dataset. To maintain full transparency and dataset integrity, every single row in my data file (train.csv, val.csv, test.csv) will include a metadata column called `is_llm_prelabeled` (True/False). I will manually review, correct, and finalize 100% of these rows myself to transform them into "gold standard" data. I will track my agreement rate with the LLM pre-labels to document how well a zero-shot model handles the initial taxonomy.

3. Failure Analysis: After evaluating both the fine-tuned DistilBERT model and the Llama-3.3 zero-shot baseline on the test set, I will compile all misclassified rows into a clean CSV format containing: Post Text, True Label, Predicted Label, and Model Confidence. I will feed this error log into an AI assistant, prompting it to analyze linguistic trends among the failures (e.g., checking if short post lengths, heavy internet slang, sarcasm, or nested parentheses correlate with model failure). I will manually pull 3-5 examples from the AI-identified error buckets to verify if the pattern is structurally sound before writing my final evaluation report.

---