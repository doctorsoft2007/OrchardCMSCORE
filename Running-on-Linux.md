Orchard 2 can be run using dotnet CLI under Ubuntu 14.04 LTS for now. Here are the steps to get it running : 

1. Install .NET Core
    * Add the new apt-get feed

        In order to install .NET Core on Ubuntu, we need to first set up the apt-get feed that hosts the package we need.

        Note: as of now, the below instructions work on Ubuntu 14.04 and derivatives. New versions are coming up soon! Also, please be aware that this feed is our development feed. As we stabilize we will change feeds where deb packages are stored.

        ```shell
        sudo sh -c 'echo "deb [arch=amd64] http://apt-mo.trafficmanager.net/repos/dotnet/ trusty main" > /etc/apt/sources.list.d/dotnetdev.list'
        sudo apt-key adv --keyserver apt-mo.trafficmanager.net --recv-keys 417A0893
        sudo apt-get update
        ```

    * Install .NET Core (dotnet CLI)

        Installing .NET Core is a simple thing on Ubuntu. The below will install the package and all of its dependencies.

        ```shell
        sudo apt-get install dotnet-dev-1.0.0-preview1-002702  //stable version
        ```
        
        * Other usefull commands
        
            This will get you a list of dotnet version that you can install
        
            ```shell
            sudo apt-cache search dotnet
            ```
            
            This will remove any installed version of the dotnet-dev runtime
            
            ```shell
            sudo apt-get remove dotnet-dev-1.0.0-*
            ```

3. Install Mono (Required by KoreBuild)

    ```shell
    sudo apt-get install mono-complete
    //or
    sudo apt-get install mono-devel //faster
    ```

4. Install Visual Studio Code
    
    If you want to be able to edit/run/debug Orchard 2 this is the IDE that you need.

    ```shell
    curl -O https://az764295.vo.msecnd.net/stable/fa6d0f03813dfb9df4589c30121e9fcffa8a8ec8/vscode-amd64.deb
    sudo dpkg -i vscode-amd64.deb
    ```

5. Get Orchard from Github repository

    ```shell
    sudo apt-get install git
    git clone https://github.com/OrchardCMS/Orchard2.git

    cd Orchard2
    sh build.sh //really important to not do "sudo" else we will make the nuget packages to be accessible by only sudo wich will cause problems with the Omnisharp installer later on.
    ```

6. Install Omnisharp

    Download with browser
    https://github.com/OmniSharp/omnisharp-vscode/releases/download/v1.0.11/csharp-1.0.11.vsix

    ```shell
    code //opens vs code from command shell
    ```

    Open the .vsix file with VS Code. It will add C# syntax highlighting and a debugger.
    