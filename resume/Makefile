# note to self: this is only intended for human development use.
# don't touch this or use it for automation.
run:
	pdflatex resume.tex
	pdflatex resume.tex
	ln -s resume.pdf resume.ja.pdf 2> /dev/null
clean:
	rm -rf resume.{aux,log,out,pdf} 2> /dev/null
	unlink -f resume.ja.pdf 2> /dev/null
