#python -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

    # Box Settings
    config.vm.box = "archlinux/archlinux"
    config.vm.box_version = "2020.03.04"

    # Box name
    config.vm.define "bde_box"

    # Network Settings
    config.vm.hostname = "archlinux"
    config.vm.network "forwarded_port", guest: 8888, host: 8888

    # Folder Settings
    config.vm.synced_folder ".", "/vagrant", disabled: true
    config.vm.synced_folder "./shared", "/home/vagrant/shared", create: true

    # Provider Settings
    config.vm.provider "virtualbox" do |vb|
        # Set a name for the machine displayed in VirtualBox
        vb.name = "bde20"
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

        # Install jdk8
        echo "Installing JDK11..."
        pacman -S --noconfirm --quiet --needed jdk11-openjdk

        # Add .local/bin to PATH
        if ! $(grep -Fxq 'export PATH="$PATH:/home/vagrant/.local/bin"' /etc/profile)
        then
            echo 'export PATH="$PATH:/home/vagrant/.local/bin"' >> /etc/profile
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
        conda install -n bde -y --quiet \
        numpy \
        scipy \
        matplotlib \
        pandas \
        jupyter \
        python-graphviz \
        pyspark \
        findspark \
        psycopg2 \
        scrypt \
        seaborn \
        plotly \
        tqdm \
        ipywidgets \
        ipycanvas \
        ipyevents \
        nltk \
        python-duckdb \
        scikit-learn \
        pandasql \
        sqlparse \
        jupyter_contrib_nbextensions
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
    config.vm.provision "spark", type: "shell", privileged: true, run: "once", inline: <<-SHELL
        # Install Apache Spark
        echo "Installing Apache Spark..."

        # Set variables
        APACHE_SPARK_VERSION=2.4.5
        HADOOP_VERSION=2.7
        SPARK_HOME=/opt/apache-spark

        if [ ! -d "$SPARK_HOME" ] || [ ! -n "$(ls -A $SPARK_HOME)" ];
        then
            mkdir -p "$SPARK_HOME"
            curl    --silent \
                    --output apache-spark.tgz \
                    "http://ftp.halifax.rwth-aachen.de/apache/spark/spark-${APACHE_SPARK_VERSION}/spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz"
            tar -xzf apache-spark.tgz -C "$SPARK_HOME" --strip-components=1
            rm apache-spark.tgz
            if ! $(grep -q "SPARK_HOME" /etc/profile)
            then
                echo "export SPARK_HOME=$SPARK_HOME" >> /etc/profile
                echo 'export PATH="$PATH:$SPARK_HOME/bin"' >> /etc/profile
            fi
        fi
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
