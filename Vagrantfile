#python -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

    # Box Settings
    config.vm.box = "archlinux/archlinux"
    # config.vm.box_version = "v20210401.18564"

    # Box name
    config.vm.define "bde_box"

    # Network Settings
    config.vm.hostname = "archlinux"
    config.vm.network "forwarded_port", guest: 8888, host: 8888 # Jupyter
    config.vm.network "forwarded_port", guest: 4040, host: 4040 # Spark
    config.vm.network "forwarded_port", guest: 7474, host: 7474 # Neo4j (currently not working)

    # Folder Settings
    config.vm.synced_folder ".", "/vagrant", disabled: true
    config.vm.synced_folder "./shared", "/home/vagrant/shared", create: true

    # Provider Settings
    config.vm.provider "virtualbox" do |vb|
        # Set a name for the machine displayed in VirtualBox
        vb.name = "bde"
        # Customize the amount of memory on the VM
        vb.memory = "2048"
        # Customize network settings on the VM
        # vb.customize ["modifyvm", :id, "--nictype1", "virtio"]
        vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
    end

    # Provisions

    # Install system packages
    config.vm.provision "root-setup", type: "shell", privileged: true, run: "once", inline: <<-SHELL
        # Update system
        echo "Updating system packages..."
        pacman -Syu --noconfirm --quiet

        # Update archlinux keyring
        echo "Updating archlinux keyring..."
        pacman -S --noconfirm --quiet --needed archlinux-keyring

        # Install development tools
        echo "Installing development tools..."
        pacman -S --noconfirm --quiet --needed base-devel git binutils tree neovim

        # Install Python, Virtualenv and Graphviz
        echo "Installing Python..."
        pacman -S --noconfirm --quiet --needed python python-pip graphviz

        # Install jdk11
        echo "Installing JDK11..."
        pacman -S --noconfirm --quiet --needed jdk11-openjdk

        # Add .local/bin to PATH
        if ! $(grep -Fxq 'export PATH="$PATH:/home/vagrant/.local/bin"' /etc/profile)
        then
            echo 'export PATH="$PATH:/home/vagrant/.local/bin"' >> /etc/profile
        fi

        # Add Apache Spark to PATH
        SPARK_HOME=/opt/apache-spark
        if ! $(grep -q "SPARK_HOME" /etc/profile)
        then
            echo "export SPARK_HOME=$SPARK_HOME" >> /etc/profile
            echo 'export PATH="$PATH:$SPARK_HOME/bin"' >> /etc/profile
        fi
    SHELL

    # Install Miniconda and create virtual environment
    config.vm.provision "miniconda", type: "shell", privileged: false, run: "once", inline: <<-SHELL
        # Install miniconda from AUR
        echo "Installing miniconda..."
        git clone https://aur.archlinux.org/miniconda3.git
        cd miniconda3
        makepkg -si --noconfirm --noprogressbar

        # Configuration and cleanup
        cd ..
        rm -rf miniconda3
        echo "[ -f /opt/miniconda3/etc/profile.d/conda.sh ] && source /opt/miniconda3/etc/profile.d/conda.sh" >> ~/.bashrc
        source /opt/miniconda3/etc/profile.d/conda.sh # add conda to path
        conda create -n bde -y --quiet python=3.7.6
        echo "conda activate bde" >> ~/.bashrc
    SHELL

    # Install Python packages in conda environment
    config.vm.provision "python-packages", type: "shell", privileged: false, run: "once", inline: <<-SHELL
        # Install Python packages
        echo "Installing Python packages..."
        source /opt/miniconda3/etc/profile.d/conda.sh # add conda to path
        conda activate bde # make sure to activate virtual environment
        conda config --add channels conda-forge

        conda install -n bde -y --quiet numpy
        conda install -n bde -y --quiet scipy
        conda install -n bde -y --quiet matplotlib
        conda install -n bde -y --quiet altair
        conda install -n bde -y --quiet vega_datasets
        conda install -n bde -y --quiet pandas
        conda install -n bde -y --quiet jupyter
        conda install -n bde -y --quiet python-graphviz
        conda install -n bde -y --quiet pyspark
        conda install -n bde -y --quiet findspark
        conda install -n bde -y --quiet psycopg2
        conda install -n bde -y --quiet scrypt
        conda install -n bde -y --quiet seaborn
        conda install -n bde -y --quiet plotly
        conda install -n bde -y --quiet tqdm
        conda install -n bde -y --quiet ipywidgets
        conda install -n bde -y --quiet ipycanvas
        conda install -n bde -y --quiet ipyevents
        conda install -n bde -y --quiet nltk
        conda install -n bde -y --quiet python-duckdb
        conda install -n bde -y --quiet scikit-learn
        conda install -n bde -y --quiet pandasql
        conda install -n bde -y --quiet sqlparse
        conda install -n bde -y --quiet jupyter_contrib_nbextensions
        conda install -n bde -y --quiet neo4j-python-driver

        pip install py2neo # conda version is outdated

        jupyter nbextension enable varInspector/main
    SHELL

    # Install and configure postgresql
    config.vm.provision "postgresql", type: "shell", privileged: true, run: "once", inline: <<-SHELL
        # Install PostgreSQL
        echo "Installing PostgreSQL..."
        POSTGRES_DATA=/var/lib/postgres/data
        pacman -S --noconfirm --quiet --needed postgresql

        # Configuration
        if [ -d "$POSTGRES_DATA" ] && [ ! -n "$(ls -A $POSTGRES_DATA)" ];
        then
            sudo -u postgres initdb -D "$POSTGRES_DATA"
        fi
        systemctl enable postgresql --now
    SHELL

    # Install and configure Apache Spark
    config.vm.provision "spark", type: "shell", privileged: false, run: "once", inline: <<-SHELL
        # Install Apache Spark
        echo "Installing Apache Spark..."
        git clone https://aur.archlinux.org/apache-spark.git
        cd apache-spark
        makepkg -si --noconfirm --noprogressbar
        cd ..
        rm -r apache-spark

        source /etc/profile
        source ~/.bashrc
    SHELL

    # Install and configure Neo4j
    config.vm.provision "neo4j-install", type: "shell", privileged: false, run: "once", inline: <<-SHELL
        # Install Neo4j
        echo "Installing Neo4j..."
        git clone https://aur.archlinux.org/neo4j-community.git
        cd neo4j-community
        makepkg -si --noconfirm --noprogressbar
        cd ..
        rm -rf neo4j-community
    SHELL
    config.vm.provision "neo4j-configure", type: "shell", privileged: true, run: "once", inline: <<-SHELL
        neo4j-admin set-initial-password h6u4%kr # needs root privileges
        systemctl enable neo4j --now
    SHELL

    # Install sqlite3kernel for jupyter notebook (needs jupyter installed, therefore not in root setup)
    config.vm.provision "sqlite3kernel", type: "shell", privileged: false, run: "once", inline: <<-SHELL
        # Install SQLite3 Kernel
        echo "Installing SQLite3 Jupyter Kernel..."

        source /opt/miniconda3/etc/profile.d/conda.sh # add conda to path
        conda activate bde # make sure to activate virtual environment

        git clone https://github.com/brownan/sqlite3-kernel.git ~/sqlite3-kernel
        cd ~/sqlite3-kernel
        python setup.py install --user
        python -m sqlite3_kernel.install

        # clean up
        rm -rf ~/sqlite3-kernel
    SHELL
end
