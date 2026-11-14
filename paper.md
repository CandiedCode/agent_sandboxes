# Paper Requirements

**Instructions:** Submit a technical paper (as a PDF) on your semester project, due **midnight, Friday Dec 11**.

**Format:** The report should be between 6-8 pages total (US letter size). The margins should be 1 inch all around, with single spaced lines and 10-12pt font size. Times font is encouraged, as is using a two-column layout (but not required). Figures should be easily readable (i.e., not too small). If helpful, Latex and Word templates for papers are available from the ACM.

Please ask if you have any questions about the format. Your goal should be to add as much content as possible while remaining clear and concise, and keeping the paper format readable.

---

**Paper Structure:** Your paper should consist of similar sections as suggested below, although feel free to tweak as suitable for your project (and ask any questions if you're unsure).

1. **Title + Authors**

2. **Abstract:** A couple paragraphs overviewing the work.

3. **Introduction:** Introduce and motivate the work (what the problem is, and why it's an important problem), and then discuss what your project does to tackle this at a high level.

4. **Related Work / Background:** Describe briefly what the relevant and prior work does, and provide any background context if necessary. Provide concrete citations of the existing work that you discuss. You are expected to have done some online literature search for relevant work, so make sure you include them.

5. **Approach:** Describe your complete project approach in technical detail, at a level where someone could reproduce it just from your descriptions. It'd be helpful to have diagrams/figures illustrating your approach, if suitable. A suggested structure is to start by clearly stating/listing your goals (and potentially challenges in achieving those goals), and then discuss your approach design and justification for your design decisions. Also acknowledge any weaknesses/limitations with your approach. If there are ethical considerations in your method design, make sure to discuss those and why your method doesn't violate ethical considerations.
   1. If you are developing an attack, carefully describe the design of your attack strategy, and justifications for your design decisions.
   2. If you are developing a defense, carefully describe the design of your defense, and justifications for your design decisions.
   3. If you are doing an empirical study (e.g., measuring or evaluating real-world security), discuss your measurement/evaluation methodology in detail (e.g., data collection or analysis process), including the reasoning behind your methodology design decisions and limitations to your evaluation/measurement (e.g., what might you not measure/observe).

6. **Findings/Results/Analysis:** Extensively and rigorously describe the outcomes or results of your approach. It'd be helpful to add graphs, tables, and any other representations of your results, with captions explaining the figures.
   1. If you developed an attack or a defense, you should discuss a careful evaluation of its effectiveness (in terms of security/privacy, and potentially along other dimensions such as performance). First describe what you'll evaluate (e.g., metrics), how you conduct the evaluation (e.g., measurement methodology), and then your findings.
   2. If you did an empirical study, discuss your data analysis strategy (in enough detail that we know exactly how you conducted your analysis) and then present your results/findings.

7. **Discussion:** Discuss the implications of your findings for security/privacy/society, potential defenses, future work, etc.

8. **Conclusion:** Short recap of what you've accomplished in your paper.

9. **Acknowledgements (if any)**

10. **References/Citations:** Feel free to use whatever citation style you'd like (although make sure to have the authors, the title, the venue/location/URL of the work, and the year of appearance). It's okay if the references/citations go beyond 8 pages (but only this section is allowed to go over length).

## Notes

- The paper must be **6-8 pages** total (US letter); only the references section may go over 8 pages.
- Section 5 (Approach) must be detailed enough to be reproduced, and must cover ethical considerations. For this project, that connects to the ethics note in [research/netsec-course-mapping.md](research/netsec-course-mapping.md): all tests run from inside our own sandboxes against endpoints and domains we own.
- Section 4 (Related Work) can draw on [research/paper-review.md](research/paper-review.md).

---

Some advice/guidance on writing technical computer science papers that may be helpful:

- <https://homes.cs.washington.edu/~mernst/advice/write-technical-paper.html>
- <https://www.froihofer.net/en/students/how-to-write-a-computer-science-paper.html>

## Grading scheme

For each section of the paper, I will assess the quality of the content, both in terms of technical completeness/correctness as well as writing quality.

- Title/Author + Abstract, Introduction, Background/Related Work, Conclusion, and References/Citations will each be assigned 5 points per section (25 points total across these 5 sections).
- Discussion will be assigned 15 points.
- Approach and Results/Findings/Analysis will each be assigned 30 points (60 points total across the two sections).

I will also assign 25 points for your project demonstrating clear hands-on investigation/experience in a network security topic (i.e., you're not simply providing a literature search or reiterating information in existing documentation/blogs/papers, but getting real-world/practical experience with network security, whether it's engaging with real-world code, data, design, or proofs).

Total: 125 points

| Component | Points |
|---|---|
| Title/Author + Abstract | 5 |
| Introduction | 5 |
| Background/Related Work | 5 |
| Conclusion | 5 |
| References/Citations | 5 |
| Discussion | 15 |
| Approach | 30 |
| Findings/Results/Analysis | 30 |
| Hands-on network security investigation/experience | 25 |
| **Total** | **125** |
