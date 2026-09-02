# Intake — the questions to ask before anything is built

This skill **leads**. It does not wait to be told the shape of the work: it asks, and it refuses
to default silently. An unanswered question becomes **a risk in the decision register with the
assumption you took written beside it** — never a default chosen quietly.

Ask in one round if you can. Ten answered questions beat thirty asked.

## A. Which entry are we at

1. Has any code been written for this system — by whom, in which tool, and is it in version
   control?
   - *Nothing yet* → this file, then the plan.
   - *Work already exists* → `retrofit.md` **first**. Do not resume forward.

## B. The specification and its oracle

2. Where do the requirements live, and is every one **numbered**? No numbering means no
   traceability, which means nothing can honestly be called done.
3. Which requirements have an acceptance oracle **outside this model** — a regulator, a human
   translator, a penetration tester, a suite you did not write? Those are Pass 2 by definition.
4. What is explicitly out of scope? Name it, or it will be built.

## C. Consequence — this drives everything else

5. Does the system hold personal data? Is there a DPIA, and what does it commit you to?
6. Does it accept file uploads from the public? → malware scanning, `security.md`.
7. Does it move money?
8. Does it publish to third parties or perform any irreversible transition?
9. **What must never happen?** One sentence each. These become gate areas, verbatim.

## D. The estate

10. Whose tenant and subscription, and who pays? The budget alert goes to their finance contact.
11. Which regions are permitted, and is residency a requirement or a preference?
12. Is this a landing zone with policy — public ingress denied, private endpoints mandated,
    tagging enforced? Ask **now**. Retrofitting private endpoints is a rebuild, not a setting.
13. Backup and retention: what does the regulation require, in days?

## E. Ownership that must move before go-live

14. Who holds the domain and DNS zone today, and who holds it at go-live?
15. Whose legal entity owns the payment account? It cannot be handed over later, only migrated.
16. Which organisation owns the repository, and who administers CI?

## F. Harnesses, models and the pass boundary — the set most often skipped

17. **Which harnesses will run this build, and which model in each?** Name them out loud.
18. Who holds the keyboard for each pass, and does the run continue unattended?
19. **Where does Pass 1 stop?** Read the operator the Pass 1 exit bar (SKILL.md §1) and get an
    explicit yes or an explicit amendment. "Almost ready for production" is not an answer.
20. **If a second harness or a cheaper model continues the work, which categories may it not
    build alone?** Read them the never-delegate list. Get it written into the plan, not agreed
    verbally in chat.

## G. Budget and pressure

21. What is the deadline, and what happens if the build misses it?
22. How much wall-clock per phase may go to gates? On the reference build, full gates were ~9% of
    run time and repeatedly earned it.
23. **If time runs short, what gets re-tiered rather than dropped?** Settle this while there is no
    pressure. Under pressure the answer is always "the gate", and that is the one answer the
    method cannot survive.

## H. The record

24. Where do the state file, decision register and traceability live, and who reads them next?
25. May decisions be taken on the operator's behalf while they are away? If yes, every one of them
    is presented on their return.

## Turning the answers into a plan

Produce these in order, and show the operator the first three **before writing any code**:

1. **The pass boundary**, with the Pass 1 exit bar restated in this project's own terms.
2. **The tier map** — every capability from set C at its tier, with the model *and the harness*
   named, recorded where the build lives rather than in a chat window.
3. **The phase list**, each phase's gate weight justified by consequence.
4. **Sprint goals as exit tests**, for the first phase only. Plan a later phase when its
   predecessor's gate returns — planning further is fiction.
