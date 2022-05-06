---
title: Tips for writing a software engineering dissertation
date: 2020-06-30 19:00:00
tags: [latex, uml]
hidden: true
---

During my university journey, I had to write many technical reports or articles about software engineering. I have some tips that helped accelerate the writing of my dissertation and may be helpful for others.

## LaTeX instead of Word

At first, I thought that LaTeX was too complex and Word was good enough. On big reports, basic tasks like inserting images, references, changing the layout, making sure everything is formatted correctly, etc., always became a hassle with Word. With LaTeX, the initial learning curve is higher, but there are templates for most kinds of documents you'll ever need, so advanced knowledge is not really required and the output PDF is much less prone to manual errors. If a local setup is not desired, try Overleaf. PowerPoint is fine for presentations, it's easy, quick and looks good. A possible alternative is reveal.js if you want to use web technologies.

## Version control

Keep your dissertation and all the project code in version control. I prefer using Git and Github to document all the work done. This way I can go back to a previous version at any time and have a backup of everything. Collaboration becomes much easier too. Just avoid including files that can be easily generated at any time (use gitignore), like the dissertation PDF or your compiled software binaries. Take care of the package management solution, all software/libraries versions should be listed. For higher reproducibility, include vendor dependencies and even data sets that are hard to obtain in an automated way. Also keep a README with instructions to compile/run and common debugging steps. Using Docker is highly recommended.

## Zotero to manage references

Nobody should be managing their references manually, simply use Zotero, which is open-source. With the browser extension, any article, book or webpage can be saved. In most cases, it will correctly extract the metadata. Combined with LaTeX, all I had to do was export the list in BibTex format, select the reference style and the bibliography section was done. Get good references from reputable journals, I like to search articles using ResearchGate and Google Scholar. It is ok to take a look at Wikipedia, but check their references and cite those instead.

## PlantUML to generate UML diagrams

With the open-source PlantUML, diagrams are defined in text, which is ideal to place in version control. In the past, I had to use tools like Visual Paradigm to create UML and BPMN diagrams. Dragging things around, specially when a diagram does not fit very well in the screen, is very time consuming, although the final result can be prettier. I would recommend [diagrams.net](https://app.diagrams.net) as an open-source alternative to PlantUML but only for more advanced use cases.

## Use your university resources

My university provided various optional lectures on topics such as LaTeX, how to do research, how to manage references and others. In addition, just by being connected to the university network, even through VPN, websites like SpringerLink and ScienceDirect will give free access to resources. There is a lot of research funded by public taxes that is behind closed gates. Keep in touch with your supervisor and schedule frequent meetings. Being a student has many advantages, from free cloud credits with the Github student pack, to the best Jetbrains IDEs.

## Visual Studio Code and high quality extensions

In my opinion, VS Code is the best text editor. It is open-source, cross-platform, reasonably fast, has integrated terminal, great git support and plenty of high quality extensions. The following extensions are a must for writing:

- [LaTeX Workshop](https://marketplace.visualstudio.com/items?itemName=James-Yu.latex-workshop)
- [PlantUML](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml)
- [Code Spell Checker](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker)

## Other recommendations

Do not leave the writing for last, do it at the same time as the project is developed, otherwise it will be way harder to stay motivated. Ask other people to read your dissertation, they will likely find flaws you haven't identified before. When doing a project in a company, seek monetary compensation, if they are cheap now, they will be cheap later. Experiment different tools and workflows to find what works best for you.
