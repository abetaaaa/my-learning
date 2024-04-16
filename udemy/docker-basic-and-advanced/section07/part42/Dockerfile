FROM ubuntu:20.04 AS base
RUN apt update
CMD ["sh", "-c", "echo My name is $my_name"]

FROM base AS development
ENV my_name=TEST

FROM base AS production
ENV my_name=Bob
