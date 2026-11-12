# Project proposal

## 1. The problem and goal

**The problem and goal in my project:**

To predict how a fire or an engine behaves, a computer program has to follow
thousands of chemical reactions at once. One program that builds such a model
automatically is RMG (Reaction Mechanism Generator). For every molecule it
meets, RMG needs to know how much energy the molecule stores (the *enthalpy of
formation*, in kcal/mol). This number decides, among other things, how fast each
reaction runs backwards.

Calculating this energy exactly takes hours of supercomputer time per molecule,
so RMG uses a shortcut called *group additivity*: it cuts the molecule into
small pieces ("groups"), looks up a stored energy for each piece in a
hand-built table, and adds them up. It also adds a surcharge for special
shapes, such as a ring of atoms bent out of its comfortable shape, which stores
extra energy like a squeezed spring.

The weak spot is the surcharge table for rings: tiny rings of three or four
atoms, rings joined to each other, and rings containing nitrogen or oxygen. It
was built by hand from limited data, and for these shapes the answers can be far
off. On 129 very accurately calculated molecules (the "Spiekermann" set), RMG is
off by about 7 kcal/mol on average, which can change a predicted balance between
two molecules by a factor of roughly 30 at 1000 K (about 730 °C). On about
125,000 neutral molecules from a large database of accurate energies (see
"Limits"), the average error is 4.9 kcal/mol with no ring, 6.2 with one ring and
8.0 with rings that share atoms.

**Goal:** improve RMG's ring surcharge table (add, split or remove entries) so
that its energy estimates match trusted reference values better.

**How improvement is measured:** the average absolute error of the enthalpy of
formation at 298 K, on molecules never used to adjust any entry and never shown
to the agent. Three sealed sets, each used once at the very end:
- a sealed part of the large database (thousands of ring molecules), which
  decides whether the agent beats the simpler methods; it is scored on all its
  molecules and on slices (see "Limits");
- the Spiekermann set (129 molecules, 70 with rings), which tests whether the
  changes help on realistic chemistry;
- the Elliott set (145 radicals, none using a ring surcharge), which only checks
  that nothing else broke.

I verified from the group files, the official fitting notebook and the
RMG-database history that neither Spiekermann nor Elliott was used to fit RMG's
tables. I looked once at RMG's *existing* errors on Spiekermann, which is how I
found that rings are the weak spot, and I applied one unrelated change to it
(filling in missing nitrogen entries) that made no difference. I have not tested
any ring change on it. A further check of the loop itself is "planted damage":
real ring entries are deleted and the agent must bring the error back down.

**What I will be able to claim, decided in advance:**
The agent against the baselines is judged on the sealed part of the large
database. Whether anything helps on realistic molecules is judged once on the 70
Spiekermann ring molecules, as an effect size with its uncertainty, not pass or
fail: a test that small notices an average gain of about 1.5 kcal/mol only about
7 times in 10, and a smaller gain rarely.
- Beats all three baselines and the gain is visible on Spiekermann: the agent
  finds better ring corrections, and I offer them to RMG's maintainers.
- Beats them on the large database but Spiekermann is inconclusive: the claim is
  limited to the large database, and transfer to realistic molecules is untested.
- Only matches the best baseline with fewer, better-explained entries: same
  accuracy, easier to understand.
- A plain refit does as well as the agent: refitting helps and no agent is
  needed.
- Nothing improves, or the improvement does not carry over: a negative result,
  reported as it is.

**Limits and requirements:**

*The data.* No new quantum-chemistry calculations. I use the "G4MP2-GDB9"
database (Narayanan et al., 2019): every possible small molecule of up to nine
heavy atoms, with energies about 0.8 kcal/mol from experiment on a test group of
459 molecules. I reduce its 133,296 entries as follows:

| Step | Left |
|---|---|
| Merge entries differing only in 3D arrangement | 128,893 |
| Keep carbon, hydrogen, oxygen, nitrogen only (drop 1,814 with fluorine) | 127,079 |
| Drop 263 already in RMG's libraries (0.2%) | 126,816 |
| Drop 133 that RMG could not process | 126,683 |
| Drop 1,586 charged structures (artifacts of the listing) | 125,097 |

A small second source is 2,285 molecules from the RMG libraries that RMG's
existing group values came from. Only 53 contain a ring, so I plan to keep them
out of the fit and use them as a small extra check.

*Tables and molecules.* RMG keeps ring corrections in two files: single rings and
rings that share atoms ("polycyclic"). Both are in scope; nothing else changes.
Only molecules with at least one ring are fitted, since only they use these
tables (the 12,681 ring-free ones cannot change). Caveat: a ring molecule's error
also includes errors of untouched non-ring groups, which get partly absorbed
into the ring entries.

*Unusual molecules.* The database lists every possible molecule, so 64% of the
neutral ones have a three- or four-membered ring, against about 2% (38 of 2,285)
in RMG's libraries. I can flag the unusual ones in advance by fixed chemical
rules written down before any agent run: a small ring sharing atoms with another
ring, two small rings, bridged or spiro rings, a ring N–N, N–O or O–O bond, or a
ring atom with a triple or two double bonds. In a quick test these flag 61% of
ring molecules (average error 8.2 kcal/mol vs 5.4 for the rest).
- I do not delete them, above all not the joined-ring ones: the polycyclic file
  can only be tested on them (60,389 molecules have joined rings, but only about
  3,700 have no small ring and no N/O ring bond). The flags only define slices
  for reporting: all molecules, ordinary-looking single-ring (about 41,000) and
  ordinary-looking joined-ring (a few thousand).
- The sealed slices are the subset test in addition to the 70 Spiekermann
  molecules. They show whether changes work on unseen molecules and on
  ordinary-looking chemistry; only Spiekermann shows whether they help in real
  RMG use. Whether to fit on everything or only on ordinary-looking molecules is
  open, and I would compare both.
- The rules are my own judgement. I know from my earlier look that Spiekermann's
  errors sit in small strained N/O rings, so I cannot claim the rules were chosen
  blind to that.

*Data access and licence.* The database is hosted by the Materials Data Facility
(DOI 10.18126/M23P9G); I downloaded it through Globus. The repository will never
contain the data, only download instructions and a script that checks the
downloaded file, so everyone fetches their own copy from the source under its
terms. The article has a non-commercial licence; I have not yet read the data's
own licence and will before final submission.

*Fixed requirements.*
- Sealed molecules are never used to fit or to choose anything. RMG-database is
  pinned to one version (commit fc7bb138f) so an update cannot leak them in.
- The agent's output is in the same format as RMG's own ring-table files, so the
  maintainers could add it to RMG-database without rewriting it by hand.

*Not decided yet* (first guesses, to settle after I see the data): how new
entry values are computed (starting point: least squares, so nobody types a
number by hand); the minimum molecules per new entry (5) and the cap on new
entries (about 10); the rule for keeping a change (the development error goes
down).

## 2. The simulator

**The simulator in my project:**

The simulator is RMG's own energy estimator (the open-source Python package
`rmgpy`), wrapped in an evaluator that I will write:

- **Input:** a proposed change to the ring tables in a simple text file (YAML):
  a ring shape to add (atoms and bonds in RMG's format) and its parent entry, or
  an existing entry to remove.
- **What it does:** applies the change, fits the best values for the new or
  changed entries by least squares on the development molecules, and
  re-estimates the affected molecules.
- **Output:** the development error with a breakdown by chemical type, the
  worst-predicted molecules with the entries they used, and whether every limit
  was respected.

It runs on my own computer, is deterministic and needs no internet. Loading the
database takes about 2.4 s and 1,000 molecules take about 27 s to process. The
full pass over the 126,683 molecules took 55 minutes, so I run it once and save
it; each proposal then re-scores only the molecules it could affect (about 1 ms
per molecule in an earlier test on a different table). Still to check: that one
proposal stays under about a minute, and how to run the tests in GitHub's
automatic checks, since `rmgpy` is hard to install. My plan is to keep the
fitting and scoring code free of `rmgpy` so most tests are fast, and to run the
few tests that need it in RMG's official Docker image.

## 3. The agent's role

**The agent in my project:**

The agent decides *which ring shapes deserve their own surcharge*. In each round
it:
1. reads the report: which molecules and ring types have the largest errors and
   which entries they used;
2. forms a hypothesis, for example "the largest errors are three-membered rings
   with nitrogen next to a double bond, which the table treats like ordinary
   carbon rings";
3. proposes a change (add a more specific entry, split one, or remove one that
   fits noise);
4. runs the simulator, keeps the change if the development error drops and the
   limits hold, otherwise undoes it and tries another hypothesis.

**What it may change:** only the *definitions* of ring shapes (atoms, bonds,
place in the table), never numbers; the simulator fits those. New entries stay
few (first guess: about 10, each backed by at least 5 molecules), using only
carbon, hydrogen, oxygen and nitrogen.

**Which molecules the agent sees:** the development molecules, i.e. most of the
large database's ring molecules, split so that families of closely related
molecules stay on the same side. A further part is sealed for judging the agent
against the baselines. Spiekermann and Elliott stay sealed until the final test.

**The for-loop question:** could a simple program get the same result? Partly,
which is why there are three baselines, each with the same number of new
entries:
- **Refit only:** keep the table's shape and recalculate the numbers of the
  existing ring entries. In a quick test this alone lowered the average error
  from 6.8 to 5.4 kcal/mol (each molecule predicted by a refit that had not seen
  it) and the errors above 20 kcal/mol from 6,341 to 2,409 out of 118,877
  molecules. Part of the gain needs no AI, and the agent has to beat it.
- **Automatic splitting:** split every badly predicted entry by every
  neighbouring atom type and keep what helps.
- **Random proposals:** a negative control.

Possible ring shapes grow very fast, and blind splitting creates many narrow
entries that fit by chance. I expect the language model to contribute chemical
judgement (ring strain, a heteroatom in the ring, joined rings), reaching a lower
error with fewer, more understandable entries. The evaluation tests this with
several agent runs and a paired comparison on the same molecules.

**A question I still need to answer:** do corrections learned from the large
database's unusual molecules help on Spiekermann's realistic ones? An early test
did not clearly help on the few realistic ring molecules available (30, error
12.0 to 11.6 kcal/mol), but that sample is too small to settle it. And will the
agent beat the plain refit? That is decided on the sealed large-database part,
since 70 ring molecules cannot separate two similar methods. If the difference is
within the noise, the honest conclusion may be that the agent mainly makes the
table easier to understand, not more accurate.

## Explain your project to someone outside your field

**My project in simple words:**

Imagine estimating how much energy is stored in a toy built from pieces. Each
kind of piece has a standard value in a table, and you add them up. For most
toys that is close enough. But where the pieces are forced into a tight loop,
they are bent like a squeezed spring and store extra energy, so the table needs
a separate correction for every kind of loop. Those corrections were written by
hand from few measurements, and for unusual loops they are mostly guesswork. A
chemistry program that simulates engines does the same with molecules: it adds
up stored values for the pieces, plus a correction for rings, which are loops of
atoms. For some small or joined rings the correction is badly wrong, and the
simulation inherits the error.

My project builds an AI assistant that improves the correction list. It looks at
which molecules are estimated worst, guesses what kind of ring the list is not
treating properly, and proposes a new entry. A separate chemistry program does
the arithmetic: it works out the best correction from a large collection of
carefully calculated molecules and reports whether predictions got better. The
assistant keeps changes that help, undoes those that do not, and repeats. At the
end I test the improved list once, on molecules the assistant never saw, and
compare it with simpler approaches that use no reasoning. I will report honestly
if they do just as well.
