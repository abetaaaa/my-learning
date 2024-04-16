FROM ubuntu:20.04

RUN touch 1.txt
WORKDIR /app/my_dir
RUN touch 2.txt
WORKDIR ..
RUN touch 3.txt

CMD ["bash"]