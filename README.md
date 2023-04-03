---
author: Christophe Gouel
---

Teaching modeling with GAMS involves practice. In my master-level class, *Introduction to General Equilibrium Modeling*, I give many exercices to the students, some that they have to do in classes and that I help them with, and some that they have to do between sessions and that they send me for grading. While the part where they work on their own to figure out solutions to their problems is important, receiving before the new class session 20 or more programs that will have to each be run separately to check whether they run as expected was something I dreaded. In this blog post, I will explain how I turn this dreaded work into something that is partly automatized and that can be done very efficiently.

To achieve this, I have been combining [GitHub Classroom](https://classroom.github.com/), [continuous integration workflow in GitHub](https://docs.github.com/en/actions/automating-builds-and-tests/about-continuous-integration), and tools for [developing in the cloud](https://docs.github.com/en/enterprise-cloud@latest/codespaces/developing-in-codespaces).

To make more concrete what I will expose, I have made available in a public repository the introductory exercise I use in my class and that I will use here as example: <https://github.com/economic-modeling-master/partial-eq-1-sector>.

# GitHub Classroom

GitHub Classroom is a service provided by GitHub that uses GitHub repositories for managing assignments in classes. It is freely available for teachers after proving an academic affiliation. For a teacher, GitHub Classroom proposes a dashboard presenting all classes and for each class a dashboard with all the assignments. An assignment is given to students by sending them a link. After clicking on the link and accepting the assignment, a repository is automatically created for them to deposit their solution in it and this repository is copied from a target repo such as [my example](https://github.com/economic-modeling-master/partial-eq-1-sector). Assignments can be done individually or in group, and in the latter case one repo is created per group. Another dashboard allows to see all students repo for a given assignment, as well as to see whether the last commit was done before or after the deadline.

From the students perspective, using GitHub Classroom just requires a GitHub account, but no installation or knowledge of Git. Without using Git, they can submit their assignments simply by uploading files manually as on any other website, in which case a commit is automatically created. So it is not limited to computer science classes and students skilled enough to learn GAMS can use it without troubles.

GitHub Classroom includes features to automatize grading. For example, by running unit tests on students code, but this does not seem adapted to GAMS programs. However, I build on similar tool to automatize the run of students solutions.

# Automatic run of students solutions

Once all students have committed their solution on their respective repositories, I can have a look at what they have done. For this part, I don't want to have to run all their GAMS programs locally. So, I have set up a continuous integration workflow in GitHub: after each commit a virtual machine is launched with a fresh GAMS install that runs all `gms` files present in the repo. After the run, all output files (`gdx`, `log`, and `lst`) are saved in a zip file for me to check as shown in this gif:

![My GitHub workflow](github-workflow.gif)

If their `gms` file does not compile, instead of a green checkmark (<span style="color:green">✓</span>) indicating compilation, there is a red cross mark (❌). In this case, I know that I have to go check their code to find the mistake, which I can also do in the cloud as explained below.

The automatic execution of GAMS is triggered by having in each repo a Yaml file with the right instructions. Here it is the file [workflow.yml](https://github.com/economic-modeling-master/partial-eq-1-sector/blob/main/.github/workflows/workflow.yml):
<details>
  <summary>Click for details of `workflow.yml`</summary>
  
```{yaml}
name: Test model solution with GAMS

env:
  GAMS_MAJOR: 29
  GAMS_MINOR: 1
  GAMS_MAINT: 0

on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install GAMS
        run: |
          cd ~
          wget -nv https://d37drm4t2jghv5.cloudfront.net/distributions/${GAMS_MAJOR}.${GAMS_MINOR}.${GAMS_MAINT}/linux/linux_x64_64_sfx.exe
          chmod 755 linux_x64_64_sfx.exe
          ./linux_x64_64_sfx.exe
          echo "~/gams${GAMS_MAJOR}.${GAMS_MINOR}_linux_x64_64_sfx" >> $GITHUB_PATH
      - name: Run GAMS
        run: |
          for gmsfile in *.gms
          do
            gams "${gmsfile}" lo=4 gdx="${gmsfile/.gms/}"
            cat "${gmsfile/gms/lst}"
          done
      - name: Archive results
        uses: actions/upload-artifact@v3
        with:
          name: gams-results-files
          path: |
            ./*.lst
            ./*.log
            ./*.gdx
```
</details>

# Fixing students errors and making feedbacks

In case of errors in their code, I could download the code to check it and modify it on my computer, but this would add a lot of frictions when having to upload back the corrected version. Instead, I am relying on [Codespaces](https://github.com/features/codespaces) which allows to start a virtual machine in the cloud. The difference with the previous virtual machine that automatically launched GAMS is that Codespaces provides a persistent machine with an editor (Visual Studio Code for the Web), a terminal to launch GAMS, and a link to the original repo to push back modifications.

GIF of codespaces

# How to deal with licensing?

One issue I encountered when developing this approach was GAMS licensing. I will tell you about the solution I adopted, but also about two other possible approaches. My solution is very simple: use GAMS 29.1. Now if you want to try GAMS without buying it, you have to request online a free demo license, but up to GAMS version 29.1 the demo version was shipped with the software and small models could be solved directly after the installation without having to provide a license. For the purpose of this class, where the students have to solve small models and, usually, do not use recently-introduced advanced GAMS features, relying on an old version of GAMS is largely sufficient.

If GAMS 29.1 is not recent enough, it is possible to use a license file. I can see at least two approaches. The first would be to store a license file in each repository of assignments and to move it automatically to where GAMS is installed on the virtual machine. Since, for my class, the repositories for exercises are all private repositories, the license would not been shared outside the class and each year, GAMS provides me with a temporary teaching license for my students. For public projects, storing the license file is not an option. In this case, the license can be stored in a [GitHub secret](https://docs.github.com/en/actions/security-guides/encrypted-secrets) and copied to GAMS folder after the installation.

# Additional benefits

Even if this setup does not require the students to learn how to use Git, it has the benefit of familiarizing them with modern development tools: GitHub, Markdown, Continuous Integration, and even development in the cloud; all skills that can be useful for modelers in and out of academia.

Another benefit of this approach is that it can be scaled up. Running the class with a hundred students would not be more difficult. It is possible to share the teacher access to GitHub Classroom with TAs that would take care of part of the load.
