## Defining Goals & Requirements
### Problem Statement

**What does this solve?**

    A cross platform, region-based-event-management, community-focused, deeni app for the community of Qaum-e-Mahdavia (mostly focused for students/youth).

**for whom?**
1. Mainly for Students' Sohbat-e-Sadiqeen.
2. Event managers who host qaum-events and seek volunteers.
3. Every Mahdavi to find Events nearby and to stay connected to bahr-e-aam updates of their respective Murshid/silsila and other deen-related updates.


### Functional Scope (v1)
MoSCoW (Must/Should/Could/Wont):

**Must**:
1. Must show nearby happening/upcoming events.
2. Must show brothers of similar age/interests nearby if the user is male, nearby sisters if female.
3. Must give bahr-e-aam/urs updates as per their Murshid, upcoming event updates via [[feat-notifs]] only (for now) and other major deeni updates like remaining days for Ramadan.
4. Must have [[feat-quran]], Hadees-e-RasoolAllah (SAWS) [[feat-hadees]] and Nuqool-e-Khalifatullah Imam Mahdi (AS) [[feat-nuqool]].
5. Must have Multilingual Support.

**Should**:
1. For event organizers, should show willing volunteers (in the same region) for events.
2. For volunteers, should show upcoming events (in the same region) to volunteer for.
3. Should have a list of Ziyarahs.
4. Should have [[feat-timeline-RasoolAllah-SAWS]].
5. Should have [[feat-timeline-Imam-Mahdi-AS]].

**Could:**
1. Could have [[feat-silsila-e-tarbiyat]].
2. Could have [[feat-silsila-e-faqiri]].

**Won't:**
1. none until now.


### Quality Attributes with numbers
1. 10k concurrent users by 1 year
2. yet to decide

### Constraints:
1. **Budget:** minimal-none
2. **Team size:** 1 for now.
3. **Timeline**: at earliest.
4. **Compliance:** Highly compliant.

**Non-functional requirements**

| Category        | Key question                       | target                                                        |
| --------------- | ---------------------------------- | ------------------------------------------------------------- |
| Performance     | Expected load, latency targets?    | 1,000 req/s                                                   |
| Scalability     | Growth expectations?               | 10k users by year 1                                           |
| Reliability     | Downtime / data-loss tolerance?    | 99.9% uptime                                                  |
| Security        | Auth, data protection, compliance? | OAuth2/JWT, GDPR                                              |
| Maintainability | Team size, tech-debt tolerance?    | team size 1. There will (not should'nt) be no debt.           |
| Cost            | Hosting budget?                    | none: everything is free tier from supabase and/or on device. |
| Observability   | How will you debug it?             | Logs, metrics, traces                                         |

**Questions worth asking:**
- What's the worst-case load pattern spiky or steady?
  - 
- What if this goes viral overnight?
- What happens if this goes down for 5 minutes? For a day? (The answer tells you whether you need real fail-over, or just a restart script.)
- Is slightly-delayed data okay, or does this need real-time accuracy?
  *  Slightly delayed data is fine unless its delayed for more than 24 hours.
- What's truly non-negotiable versus nice-to-have right now?