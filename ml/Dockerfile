FROM kserve/sklearnserver

COPY model /tmp/model

CMD ["--model_dir", "/tmp/model", "--model_name", "wine-rater"]