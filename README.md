# 📑 reviewer - Automated feedback for economics research papers

[![](https://img.shields.io/badge/Download-Reviewer_Software-blue.svg)](https://github.com/joshunspeakable173/reviewer)

This tool helps researchers process academic economics papers. It uses a series of automated agents to read a PDF file and generate a structured review. The software handles the extraction of text, the analysis of content, and the organization of feedback.

## 🛠 Prerequisites

You need a computer running Windows. This software requires a small amount of disk space to save academic papers and the resulting analysis files. You do not need to be a developer to use this tool. Before you start, ensure you have a copy of the PDF file you wish to review.

## 📥 Downloading the Tool

Visit the repository page to download the software files to your computer.

[Download the Reviewer software here](https://github.com/joshunspeakable173/reviewer)

Look for the button labeled "Code" on the page and select "Download ZIP" from the menu. Save this folder to your desktop or your documents folder. Extract the contents of the ZIP file by right-clicking the folder and choosing "Extract All."

## ⚙️ Setting Up Your Environment

Open the folder you extracted. This folder contains the necessary scripts to run the review process. The software functions through a command-line interface. 

To prepare the software, search your computer for "PowerShell" and open the application. Navigate to the folder you extracted by typing:

cd C:\Users\YourName\Desktop\reviewer

Replace "YourName" with your actual computer username. This command tells the computer to look inside the folder you saved.

## 🚀 Running Your First Review

Place the PDF file of the economics paper you want to review into a subfolder named "inputs" inside your main reviewer folder. If the folder does not exist, create it. 

Once your file sits inside the inputs folder, return to the PowerShell window. Type the following command to start the analysis:

.\.venv\Scripts\python.exe scripts\review_paper.py --pdf "inputs\your_paper_name.pdf"

Replace "your_paper_name.pdf" with the actual name of your file. The system will now perform several steps. First, it converts the PDF into a format the internal agents understand. Then, it creates specific prompts for the review process. 

The software checks the quality of the converted text before it begins the substantive review. If the quality appears low, the software attempts to repair the text automatically. Finally, the system selects relevant reviewers to provide feedback on your document.

## 🗂 Understanding the Output

The software organizes all results in a folder named "work" inside your main reviewer directory. Each paper receives its own unique folder based on its title. 

Inside your paper's folder, you will find several subdirectories:

1. Parsed: This folder holds the raw text version of your paper.
2. Prompts: This folder contains the instructions the system sent to the artificial intelligence agents.
3. Reviews: This folder stores the final documents containing the expert feedback.

You can open these files using any standard text editor or a word processor to read the generated commentary.

## 🔍 Troubleshooting Common Issues

If you receive an error when running the command, check these items:

* Verify that you typed the file path correctly in the PowerShell window.
* Ensure the PDF file exists in the "inputs" folder.
* Confirm that your internet connection remains stable, as the tool connects to remote services to process the reviews.
* Check that you extracted all files from the ZIP folder before execution.

The software assumes the paper follows standard economics academic formatting. If the paper uses non-standard fonts or unusual layout structures, the parser might require more time to process the content. 

The system performs a preflight check on every run. If the check reports issues with the document text, the tool triggers a repair agent to fix the document structure. This ensures the best possible review quality. You do not need to adjust settings for this process, as the tool handles the selection of reviewers automatically based on the content of the paper.