# Don Alejandro of La Mancha Campus and the Windmills; or, Ted from Connecticut's Excellent Time-Travel to the Court of King Stonebraker, wherein a Database Problem is Solved by Calling Things for What They Do

Once upon a time, in the mist-shrouded Faculty of Sorcerous Arts at Berkeley, there lived a powerful mage king named Stonebraker. He was not the eldest of the mage kings, nor the most celebrated in the merchant courts — the Oracle King had claimed that title years before, with his circular undo scrolls and his armies of licensed consultants. But Stonebraker had done something none of the others had managed. He had devised time-travel magic.

Not the parlour trick of prophecy nor the crude rewinding favoured by middle management. True time travel. His system — called POSTGRES, in the fashion of mages who name things in capital letters — never destroyed the past. Every change to every record produced a new version, and the old version remained, physically present, addressable, queryable. A scholar could ask: show me this table as it existed at sunrise on the third day of the month. And the system could answer, because every version it had ever produced was still there, sitting in the heap like sediment in a riverbed.

A daemon named the vacuum cleaner swept the oldest sediment off expensive storage onto archival media. The name fit. The job was cleaning up after time travel.

Then the mage king moved on to other projects, as mage kings do, and his apprentices removed the time travel magic. They kept the storage architecture because the same no-overwrite property that made time travel possible also made concurrent access work, letting multiple scholars read consistent views of the same records without blocking each other. The vacuum cleaner stayed as well growing with extra responsibilities like a djinn ballooning from a bottle.

Forty years on, the removed magic stirred one last time with a stranger arriving at the mage king's court.

---

The traveller materialized  on a skateboard at the mage king's court, wearing a t-shirt that said WYLD STALLYNS, and a lanyard with an ID badge that read TED, DBA — INFRASTRUCTURE. He rolled across the marble floor, dragged his back foot to a stop, kicked the board's tail to flip it up into his hand, and looked around.

"Whoa," he said.

The mage king, seated at a table covered in POSTGRES documentation and what appeared to be a half-eaten burrito, looked up.

"Who are you?"

"Ted, your most excellent majesty. Database administrator. From the future. Well — your future. My present. It's a whole thing."

"How did you get here?"

"Your time-travel feature, dude. The one you built into POSTGRES. I mean, everyone says it doesn't exist anymore, but—" Ted gestured vaguely at himself, standing in 1986. "Evidence suggests otherwise."

Stonebraker put down the burrito. "You came from the future to tell me something about POSTGRES?"

"About your vacuum cleaner, specifically. See, in my time, there's this dude — Don Alejandro, head of IT at this university campus that runs your database. And the campus is in crisis. Like, totally bogus crisis. Teenagers camping in a tent village outside the gates because the registration office can't issue new room numbers. And Don Alejandro is convinced it's the power supply."

"The power supply."

"The windmills. Campus is energy self-sufficient, right, big beautiful windmills on the hill. Don Alejandro is on his bicycle brandishing a screwdriver, headed for the windmills like a medieval knight on a rickety old horse with a piddling little lance, and me on my skateboard trailing behind like his squire yelling at him about the windfarm not being a gang of monstrous evil giants, and the great Don, bicycle slipstream ruffling his thinning mane, yelling back: 'the infrastructure is exhibiting non-deterministic latency characteristics consistent with intermittent power fluctuations.'"

"And it's not the power supply."

"Dude. It is *so* not the power supply. Can I tell you the story?"

The mage king pushed aside the documentation. "Tell me the story."

---

So there's this campus, right? Big place. Multiple districts — undergrad housing, physics department, arts faculty, the whole deal. Every district has buildings, every building has rooms, every room has tenants. The tenants are researchers, teachers, students, all kinds of people with all kinds of stuff — one guy's got a laptop on a beanbag, the next one's got a full workstation and a rack of equipment and needs an entire wall for her whiteboard.

Now here's the thing your majesty built into the system and which is still totally running under the hood even though nobody can see it anymore: when a tenant moves out, their belongings stay in the room. When a tenant gets reassigned to a new room — new equipment, updated setup, whatever — the old stuff stays in the old room. Nobody overwrites anything. Nobody clears anything automatically. It all just... accumulates.

"That's the no-overwrite storage model," said Stonebraker.

"Exactly, dude. And it's excellent, because it means everybody walking through the building sees a consistent picture. Doesn't matter when you walked in — the stuff that was there when you started your tour is still there. Nobody moved anything out from under you. That's your MVCC right there, working perfectly, forty years later."

"But the rooms fill up."

"The rooms *totally* fill up. Which is why you invented the vacuum cleaner — the housekeeper. They go room by room, clearing out the belongings left behind by tenants who moved out or left campus. Room's clean, space is available for the next tenant. Beautiful."

"So what went wrong?"

"Nothing went wrong, exactly. It's more like... the housekeeper kept getting extra jobs."

---

Ted sat on the edge of the mage king's table, skateboard across his knees.

"OK so the head of housekeeping manages a pool of them — like three by default — and she dispatches them to whichever district needs the most attention. So a housekeeper shows up at a building, and they don't just clear rooms anymore. They do four things.

"First: room turnover. That's the original gig. Clear out belongings left by departed tenants, update the building's free-space registry so the front desk knows where to put new arrivals. This is the job you built the vacuum cleaner for. Still the one most people think of when they hear the name.

"Second: tenant census. While they're visiting rooms, they take a sample of who's living there. Disciplines, demographics, how clustered they are by floor. The front desk uses this to answer questions efficiently — someone comes in asking where to find a Malawian physicist or an Armenian linguist, the front desk consults the census and picks the best route instead of sending someone to knock on every door. Census goes stale as people move in and out. If it hasn't been refreshed, the front desk starts sending visitors the long way around.

"Third — and this is the gnarly one, your majesty — permanent registration stamping."

"Explain."

"Every tenant who arrives on campus gets a temporary registration number from a campus-wide pool. The pool is finite. Thirty-two bits, so about four billion numbers. The numbers go up: one, two, three, four. But the counter is circular. After four billion, it wraps back to zero. And the way the system figures out whether a registration is old or new is relative — it measures the distance around the circle. Anything less than two billion ahead of the current number is 'in the future.' Anything behind is 'in the past.'

"So a tenant registered as number five hundred, when the counter's at a thousand — clearly in the past. Fine. But when the counter's gone all the way around and it's at three billion, number five hundred is now almost three billion *ahead*. Which looks like the future. The campus shuts down new registrations before it ever gets that confused. Visitors can still walk in and ask questions — the front desk works fine — but no new arrivals, no room changes, no move-outs. The campus goes quiet."

Stonebraker was quiet for a moment. "That's a consequence of the thirty-two-bit counter."

"Totally. And the fix is: the housekeeper goes around and stamps tenants permanent. They take their temporary number and replace it with a marker that just means 'old, don't compare, always in the past.' That tenant is out of the numbering game forever. Can never wrap around. The housekeeper does this for every tenant who's been around long enough — fifty million transactions by default, which is a ton of margin.

"But here's the thing. Every building tracks how far along it is — what's the oldest unstamped number still in the building. That's the building's deadline. And every district tracks the worst deadline across all its buildings. And the campus tracks the worst across all districts. One stuck building holds back the whole campus."

"And that's what happened."

"That is *exactly* what happened. But I haven't told you about job number four yet."

---

"Fourth job: status board updates. Every building has a board in the lobby with one entry per room. Two flags. First flag says 'settled' — all tenants in this room are accounted for, no departed tenants' stuff to clear. Second flag says 'completed' — all tenants in this room have been stamped permanent.

"The status board has two customers. The next housekeeper who visits can skip rooms marked settled — nothing to do there. And the front desk can use it for a shortcut: if someone asks about a tenant, and the filing cabinet says she's in room 304, and the board says room 304 is settled, the front desk can answer without sending anyone upstairs. Filing cabinet plus settled flag equals instant answer. That's your index-only scan right there."

"You'd need a per-page bitmap for that," said Stonebraker. "One bit per page. All tenants visible or not. Call it a visibility map."

"Exactly, dude. And it matters because without it, the front desk has to send someone to the room every single time, even when the filing cabinet already has the answer. The status board is what makes the filing-cabinet-only path work. And if the housekeeper falls behind and rooms lose their settled flag — because any change in a room clears both flags — then the front desk is back to running upstairs for every question. Performance drops and nothing in the question itself changed. Just the board went stale."

---

"So now you've got the four jobs," said Ted. "Room turnover, tenant census, registration stamping, status board. One housekeeper, one visit, all four. Efficient. But—"

"But they're unrelated."

"Completely unrelated! Different triggers, different failure modes, different stakes. And three of them are called VACUUM. The census is actually a separate command — ANALYZE — that also gets run by the same daemon, which means the housekeeper sometimes takes a census and sometimes doesn't depending on what the head told them to do. Four jobs, two command names, one daemon. Naming made total sense when you built it—" Ted gestured around the court. "When the vacuum cleaner cleaned and that was it. But dude, imagine calling a person who cleans rooms, takes a census, stamps registrations, AND maintains a status board a 'vacuum cleaner.' You'd think they only cleaned."

"What would you call it?"

"Housekeeping. Just... housekeeping."

---

"OK so here's the crisis," said Ted. "Freshman season. Thousands of new tenants arriving. Every corridor packed with people carrying furniture, moving boxes, swapping rooms. The housekeepers are physically present in the buildings but they can barely move — they're throttled, working slow on purpose so they don't make the congestion worse by competing for corridor space.

"Meanwhile, totally separate problem, totally different building, there's this one researcher on a publication deadline. She's been working for weeks, running around the building consulting colleagues in every office — needs to check someone's notes on the third floor, verify something in a lab on the fifth, back to her own office, then back to the third floor again. If anyone clears or moves anything in any room she might visit next, her work breaks. She needs the whole building exactly as it was when she started.

"So the housekeeper in that building is stood down. Not because the corridors are busy — that building's quiet. Because one person's unfinished work means nothing can be touched anywhere she might go, and she might go anywhere.

"The housekeeper can still stamp tenants permanent in that building — that doesn't move or change anything. But they can't clear departed tenants' belongings. And if they can't clear their belongings, their old registration numbers are still on file in the building. And the building's deadline — the oldest unstamped number — can't advance past those stuck numbers. Even though every living tenant has been stamped, the departed ones are holding back the count."

"Because you can't advance the building's horizon past records you haven't cleared."

"Right! The stamping is happening but the deadline isn't moving. And that building's stuck deadline holds back the district, and the district holds back the campus, and the campus registration office runs out of numbers and—"

"Tent village."

"Tent village, dude. Teenagers camping at the gates. Can't issue new registrations. The whole campus locked up because of one researcher's deadline in one building in one district that Don Alejandro has never even visited."

---

"And Don Alejandro—"

"Don Alejandro is on his bicycle. With a screwdriver. Headed for the windmills. Because he looked at the symptoms — registration office not issuing numbers, buildings slow, everything backing up — and he concluded that it must be the power infrastructure. 'The windmills,' he said — and I am not making this up — 'are exhibiting sub-nominal rotational dynamics inconsistent with the manufacturer's baseline specifications.'

"I'm on my skateboard going 'Dude. Dude. It's building 7 in the physics district. There's a researcher. She's got a deadline. We just need to talk to her.' And he's like—" Ted adopted a nasal, officious voice — "'Theodore, I have been administering campus infrastructure for twenty-three years and I can assure you that power supply fluctuations are the root cause of ninety percent of systemic anomalies.'

"And he's also in a hurry because he's got a dinner reservation with Dulcinea."

"Who is Dulcinea?"

"She runs housekeeping. Don Alejandro has been trying to take her to dinner for about six months. She keeps agreeing and he keeps cancelling because of infrastructure emergencies. Tonight he is *not* cancelling. So he wants the windmill theory to be correct because it's a quick fix — tighten some bolts, reset a breaker, done, dinner. The alternative — that the problem is a stuck building in a district he'd have to bike across campus to reach, involving a researcher he'd have to negotiate with, requiring him to understand the registration stamping mechanism he's never bothered to learn — that alternative means no dinner."

"So let me get this straight. The whole campus is jammed because of a housekeeping problem. And the woman he's skipping the diagnosis to have dinner with—"

"... runs housekeeping. Dude, the answer is literally his date. He could just ask her over the wine. Instead he's on a bike. With a screwdriver."

"So the incentive is to not understand the problem."

"The incentive is *absolutely* to not understand the problem. The problem is simple but the explanation is long and the fix requires going to the right building instead of the spectacular. Building 7 in the physics district is not impressive like the windmills. Don Alejandro, dude, is frame-locked on his Dulcinea date — and she's not going to be charmed by 'sorry, I had to go talk to a researcher about her publication deadline.'"

"Has he ever asked her about her work?"

"Your majesty. Don Alejandro does not ask Dulcinea about housekeeping. Don Alejandro recites her a poem he wrote about the windmills."

---

"How does it end?" asked Stonebraker.

"I don't know yet, dude. I came here to ask your data mage's highness for help."

The king was quiet for a while. Then he picked up a pen and crossed out VACUUM on the document in front of him. Above it he wrote HOUSEKEEPING.

"Room turnover," he said, writing. "Tenant census. Registration stamping. Status board." He looked up. "You said the status board has two flags — settled and completed. You'd want a per-page bitmap for that. One bit for all-visible, one bit for—"

"All-frozen. Yeah. That doesn't get invented until 2009. The first bit, I mean. The second one, 2016."

"Two bits per page," said Stonebraker, and wrote it down like it was obvious.

"You're kind of terrifying, your majesty."

"Go home, Ted."

---

Ted landed on his skateboard at the campus gates amidst a throng of students walking through towards the registration office, carrying boxes, in sunshine. When he'd left it had been raining and there had been a tent village. 

Riding his board along tree-lined campus paths, he passed building 7 in the physics district. Stopping to check the situation, he found a housekeeper inside, working room by room, unhurried. The status board in the lobby was almost entirely green. The researcher's office was empty and quiet — deadline met, paper submitted, session closed.

Don Alejandro on the restaurant patio beyond the campus gates was sitting across from Dulcinea laughing at something. Don Alejandro's shirt was clean. No grease. No screwdriver in his pocket. On the table between them, next to a half-empty bottle of wine was a thick manual with a title Ted had never seen before: *Campus Housekeeping: A Guide to Room Turnover, Tenant Census, Registration Stamping, and Status Board Maintenance.*

Don Alejandro saw Ted rolling past and gave him a small nod, the kind that means: everything is under control and always was.

Windmills on the hill kept turning.

---

*The campus is a cluster, the districts are databases, the buildings are tables, the rooms are 8KB heap pages. The tenants are tuples. The filing cabinets are indexes. The front desk is the query planner. The housekeeper pool is autovacuum. The head of housekeeping is the autovacuum launcher. The status board is the visibility map. The registration numbers are transaction IDs. The stamping is freezing. The tenant census is ANALYZE — a separate command that the autovacuum daemon also triggers, on its own schedule, with its own thresholds. The researcher on a deadline is a long-running transaction. The tent village is read-only safety mode to prevent wraparound.*

