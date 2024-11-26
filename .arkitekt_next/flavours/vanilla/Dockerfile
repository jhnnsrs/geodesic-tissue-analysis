FROM mathworks/matlab:r2024b

COPY App.m /home/matlab/
COPY .arkitekt_next /home/matlab/.arkitekt_next
COPY app.py /home/matlab/

RUN pip install "arkitekt-next[all]==0.8.46"
