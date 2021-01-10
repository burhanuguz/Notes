# Change Language of a Debian Based Image
> This is not a wanted thing. If it is possible just only use English.
> This solution is used in my old company where they were Turkish **characters** and because of that they could not compile. 
- Dockerfile can be created like below. 
```bash
FROM ubuntu
 
RUN apt-get update && \
    apt-get install -y locales language-pack-tr language-pack-tr-base && \
    rm -rf /var/lib/apt/lists/*

ENV LANG=tr_TR.UTF-8
ENV LANGUAGE=tr_TR:tr
ENV LC_ALL=tr_TR.UTF-8

ENTRYPOINT ["/bin/bash"]
```
> The reason why we entered it as **ENV** is that, when we change the **USER** for security reasons we wanted to keep language as Turkish. 
