# Bacterial survival and morphological heterogeneity

This repository contains tables and jupyter notebooks needed to reproduce analysis and generate all plots from main and supplementary figures from the article "Extending digital biology: bacterial survival and morphological heterogeneity under antibiotic stress" by Maikranz et al.
For full functionalty there is also a zenodo archive with imaging files and a sql database associated. Furthermore, we provide a docker container that allows to run
a flask webserver which easly allows to visualise images and the associated labels. This also provides access to the labelling interface that allows context aware and context free annotations. Lastly, several scripts that produce intermediate tables depend on the webserverer running. If so this is mentioned inside the scripts.


# External Data

Because of the size of the imaging files and the sql database it is provided via zenodo  https://doi.org/10.5281/zenodo.17193390

# Running the webserver in Docker container
1. Download image data and sql database from zenodo into a folder (for instance /home/your_username/). Make sure the database is at the same level as the folder of the individual antibiotics. 
2. Download docker image from github repo 
   ```bash
   docker pull ghcr.io/baroudlab/adclab:latest
   ```
4. Activate docker on your computer (see https://docs.docker.com/engine/install/ for installation)
5.
```bash
   docker run -p 8080:5000 -v /home/your_username/:/home/default-user/data/ ghcr.io/baroudlab/adclab:latest
   ```
6. Access the application at: http://localhost:8080


#  Brief explanation of the scripts not associated to figures
1.  Construct table with training dataset from the webserver and create  tif-crops used for training
2.  Trains pytorsch's resnet34 on the training data
3. Applies trained neural network to all data and constructs tables with labels
4. Extracts single morphology probabilities from tables
5. Extracts coocurance morphology probabilities from tables

# Description of other data provided
1. Contains the weights of the trained neural network
2. Tables/ contains all the tables used in the figures

