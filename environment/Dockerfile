FROM mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04

USER root:root

# Install NFS stack
RUN apt-get install -y nfs-common fio

# Install Python dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt && rm requirements.txt

CMD ["bash"]