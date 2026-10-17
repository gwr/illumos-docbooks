
Install Python (brew install python)

Download and install MacTex, eg. from:
https://tug.org/mactex/mactex-download.html

The binaries can then be run from:
/Library/TeX/texbin/latex
PATH=$PATH:/Library/TeX/texbin/latex

--------------------

The Homebrew package names needed are:

brew install make maven imagemagick python
brew install --cask mactex

They provide:

• make :  gmake
• maven :  mvn
• imagemagick :  mogrify
• python :  runtime needed by dblatex
• mactex :  pdflatex, epstopdf, and related TeX tools

dblatex is not currently in Homebrew core. This repository has the custom dblatex.rb formula, though it references the older dblatex 0.3.7 release; the installed version here is 0.3.11. ghostscript is also available as a separate formula if MacTeX does not provide gs:

brew install ghostscript


--------------------

# make -d pdf BOOKS=smb-admin
...
Must remake target `build/smb-admin/smb-admin-print.pdf'.
dblatex -p src/dblatex/params.xsl -P latex.page.size=letter -o build/smb-admin/smb-admin-print.pdf raw/smb-admin/smb-admin.book
