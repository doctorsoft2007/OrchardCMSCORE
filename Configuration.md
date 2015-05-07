# Configuration Design

* Move all configuration files to use the new IConfiguration interface.
* All configraiton files can be specified using the following order and extension
  * .txt (Default Yaml Implementation)
  * .json (.AddJson())
  * .xml (.AddXml())
  i.e. module.txt, module.xml, etc.

Module.txt
Settings.txt
Theme.txt
Config files.