GRR is now available from
[pip](https://pip.pypa.io/en/stable/installing/). If you’re looking for
an install that sets up system daemons and everything ready for
production, use the deb package via the [quickstart](quickstart.md)
instructions. Use these instructions to set up a dev environment or
upgrade a server to a newer version from github.

The packages are:

  - Server:
    [grr-response-server](https://pypi.python.org/pypi/grr-response-server)

  - Client:
    [grr-response-client](https://pypi.python.org/pypi/grr-response-client)

  - Client templates and components:
    [grr-response-templates](https://pypi.python.org/pypi/grr-response-templates)

To install the version released on pypi (the deb and rpm packaging stuff
is only needed for the server to repack the client installers). Note
that you **must** upgrade pip and run in a virutalenv for the install to
work:

    sudo apt-get install -y \
    debhelper \
    dpkg-dev \
    libssl-dev \
    prelink \
    python-dev \
    python-pip \
    rpm
    
    sudo pip install --upgrade pip
    sudo pip install virtualenv
    
    virtualenv GRR_ENV
    source GRR_ENV/bin/activate
    pip install grr-response-server
    pip install --no-cache-dir -f https://storage.googleapis.com/releases.grr-response.com/index.html grr-response-templates
    
    # Then run initialize as normal
    grr_config_updater initialize

Installing the client for development is just as easy thanks to
pre-built wheels hosted on pypi for OS X and Windows:

    virtualenv GRR_ENV
    source GRR_ENV/bin/activate
    pip install --pre grr-response-client

# Installing GRR server for dev, i.e. tracking HEAD

Installing from pip is great for development because you can have
multiple versions installed on the same machine without having them
conflict with each other. When you use pip "--editable" mode your
installed version will be linked to your src tree so you can git clone
from github, install, edit, and have any changes reflected immediately
in your installed server.

Even if you’re not doing development you may want to upgrade your GRR
server to use a commit from github to grab a bugfix or a new feature
that isn’t in a release yet. Once you have installed the deb (note it
needs to be 3.1.0-rc2 or newer):

    sudo apt-get install -y \
      debhelper \
      default-jre \
      dpkg-dev \
      git \
      libffi-dev \
      libssl-dev \
      prelink \
      python-dev \
      python-pip \
      rpm \
      wget \
      zip
    
    cd /usr/local && \
      wget --quiet "https://github.com/google/protobuf/releases/download/v3.3.0/protoc-3.3.0-linux-x86_64.zip" && \
      unzip -q protoc-3.3.0-linux-x86_64.zip -x readme.txt && \
      rm protoc-3.3.0-linux-x86_64.zip
    
    sudo pip install --upgrade pip
    sudo pip install virtualenv
    
    git clone https://github.com/google/grr.git
    virtualenv GRR_NEW
    source GRR_NEW/bin/activate
    # This sets up NodeJS inside the virtualenv.
    # NodeJS is needed to compile AdminUI JS/CSS files.
    pip install nodeenv
    nodeenv -p --prebuilt
    source GRR_NEW/bin/activate
    # This installs GRR.
    cd grr/
    pip install --editable .
    pip install --editable grr/config/grr-response-client/
    pip install --editable grr/config/grr-response-server
    pip install --no-cache-dir -f https://storage.googleapis.com/releases.grr-response.com/index.html grr-response-templates
    pip install --editable api_client/python
    pip install --editable grr/config/grr-response-test

Then edit /etc/default/grr-server and set:

    GRR_PREFIX=/path/to/GRR_NEW
