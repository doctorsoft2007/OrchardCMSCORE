## Configuration Design

At present, Orchard uses a lot of configuration files, theme.txt, module.txt, hostconfiguration.config etc etc. With the introduction of a configuration pipeline within VNext, we should also look to move towards this.

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