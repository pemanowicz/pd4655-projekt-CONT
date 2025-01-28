# pd4655-projekt-CONT
# Instrukcja Projektu

```bash
# 1. Stwórz katalog dla projektu
mkdir pd4655-projekt-CONT

# 2. Wejdź do tego katalogu
cd pd4655-projekt-CONT

# 3. W katalogu utwórz pliki
touch docker-compose.yml
touch Dockerfile
touch nginx.conf

# 4. Utwórz katalog input i umieść tam plik FASTQ
mkdir input
mv ~/Downloads/SRR8786200_1.fastq.gz input/

# 5. Instalacja narzędzia FastQC
sudo apt update
sudo apt install fastqc

# 6. W pliku Dockerfile umieść instrukcję zbudowania obrazu z narzędziem FastQC
cat > Dockerfile <<EOL
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \\
    openjdk-11-jre-headless unzip perl wget && \\
    wget https://www.bioinformatics.babraham.ac.uk/projects/fastqc/fastqc_v0.12.1.zip && \\
    unzip fastqc_v0.12.1.zip && \\
    chmod +x FastQC/fastqc && \\
    mv FastQC /usr/local/ && \\
    ln -s /usr/local/FastQC/fastqc /usr/local/bin/fastqc && \\
    apt-get clean && rm -rf /var/lib/apt/lists/* fastqc_v0.12.1.zip

ENV JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
ENV PATH=\$JAVA_HOME/bin:\$PATH
ENV CLASSPATH=/usr/local/FastQC:/usr/local/FastQC/htsjdk.jar:/usr/local/FastQC/jbzip2-0.9.jar:/usr/local/FastQC/cisd-jhdf5.jar

ENTRYPOINT ["fastqc"]
CMD ["--help"]
EOL

# 7. Ściągnij obraz nginx:latest
docker pull nginx:latest

# 8. W pliku nginx.conf umieść konfigurację dla nginx
cat > nginx.conf <<EOL
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index SRR8786200_1_fastqc.html;

    location / {
        try_files \$uri \$uri/ =404;
    }
}
EOL

# 9. W pliku docker-compose.yml zdefiniuj oba kontenery, ich sieć i wolumeny
cat > docker-compose.yml <<EOL
version: '3.8'

services:
  fastqc:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: fastqc_container
    volumes:
      - fastqc_results:/results
      - ./input:/input:ro
    networks:
      - fastqc_network
    environment:
      JAVA_HOME: /usr/lib/jvm/java-11-openjdk-amd64
      CLASSPATH: /usr/local/FastQC:/usr/local/FastQC/htsjdk.jar:/usr/local/FastQC/jbzip2-0.9.jar:/usr/local/FastQC/cisd-jhdf5.jar
    command: /input/SRR8786200_1.fastq.gz -o /results

  nginx:
    image: nginx:latest
    container_name: nginx_container
    volumes:
      - fastqc_results:/usr/share/nginx/html:ro
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    ports:
      - "8080:80"
    networks:
      - fastqc_network

volumes:
  fastqc_results:

networks:
  fastqc_network:
EOL

# 10. Uruchom projekt za pomocą komendy
docker compose up --build

# 11. Sprawdź zawartość folderu input
ls input/

# Oczekiwany wynik:
# SRR8786200_1_fastqc.html
# SRR8786200_1_fastqc.zip

# 12. Przeprowadź analizę pliku FASTQ
fastqc -o input/ input/SRR8786200_1.fastq.gz

# 13. Upewnij się, że:
# - Kontener FastQC analizuje pliki FASTQ i zapisuje wyniki w wolumenie
docker run --rm -v $(pwd)/input:/data staphb/fastqc fastqc /data/SRR8786200_1.fastq.gz

# - Kontener NGINX hostuje wyniki pod adresem:
# http://localhost:8080/SRR8786200_1_fastqc.html
