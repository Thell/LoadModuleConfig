LoadModuleConfig PowerShell Module
==================================

Overview
--------

Provides simple control for saving and loading configuration files for Powershell
modules using CliXml.  Distribute your module with this script and it will
generate an xml based configuration file in the user's local AppData directory
in a WindowsPowerShell\Modules\$ModuleName\$ModuleName.config.xml file and will
store and populate the configuration data in a Global:[hash] variable with the
name $($ModuleName)Settings.

This is a fairly automated config generator, loader, and exporter that invokes
during module import.  Unlike the PrivateData field in a module manifest file
which is tied to the module itself, this solution was created for storing both
user and project settings that would not likely change with a versions of the
module itself.  Perhaps values that would be tedious to pass to functions again
and again.

Instead of doing xml creation and parsing manually it uses PS built-in CliXml.

Reasoning
---------
Looking around the internet it is easy to find multiple methods of loading and
using configuration data for scripts, not as easy to find examples for PowerShell
scripts, and even rarer sill for modules.  Yet, in each that was found it seemed
that something was missing.  Using a plain text file to store key/value pairs
isn't very PowerShelly, and using an xml file where you need to create the xml
and create the writer/loader wasn't a good option either.

Usage
-----

---

###_Setup_:

- Use `New-ModuleManifest` to create a manifest named
the with the same name as the manifest you are loading data for but with an
extension of 'config.psd1'

- Enter LoadModuleConfig.ps1 in the NestedModules field.

#####_This is what starts the loader. (As seen in the FuncGuard module):_

	# Modules to import as nested modules of the module specified in ModuleToProcess
	NestedModules = @('FuncGuard.config.psd1')

- Open the new manifest in an editor and change the PrivateData field to include
hashes for LocalUser and Settings.  After the Settings key add any values that
you want to include with your module. _This data will be your module's default config data._

#####_Here is an example PrivateData field (as used in the FuncGuard module):_

	# Private data to pass to the module specified in ModuleToProcess
	PrivateData = @{
		"LocalUser" = @{
			"Settings"=@{
				"ActiveConfiguration" = "PSUnit"
				"Configurations"=@{
					"PSUnit" =@{
						"Name"="PSUnit";
						"Path"=$(join-path -Path $Env:Temp -ChildPath "FuncGuard.PSUnit");
						"Prefix"="PSUNIT"
					}
				}
			}
		}
	}

- After loading the module a global variable will be available with a name of
`$($ModuleName)Settings` and the values from the `Settings` section of the
PrivateData can be accessed using normal hash retrieval.

###_Using the settings hash (as used in the FuncGuard module):_

	$config = $FuncGuardSettings["Configurations"].$($FuncGuardSettings["ActiveConfiguration"])
	$Path = $config.Path

- To save settings that have been modified simply call the export() method.

###_Modifying and saving a setting (as used in the FuncGuard module):_

		if ( $FuncGuardSettings.("Configurations").containskey($Name) -eq $false ) {
			$Configuration=@{"Name" = $Name; "Path" = $Path; "Prefix" = $Prefix; "Type" = "Custom"}
			$FuncGuardSettings.("Configurations").add($Name, $Configuration)
			$FuncGuardSettings.export()
		}


### Parameter validation and default value setting.
The FuncGuard module manages various projects, where one is set to 'active', yet
the active value is just a provider for default param values.

	[Parameter(ParameterSetName="Validate", Mandatory=$true, Position=1,
		HelpMessage="`nName used to identify the project within the configuration settings.")]
	[Parameter(ParameterSetName="Force", Position=0, Mandatory=$true,
		HelpMessage="`nName used to identify the project within the configuration settings.")]
	[ValidateScript({
		($FuncGuardSettings.("Configurations").containskey("$_") -eq $false) -or
		($Force)
	})]
	[string]
	$Name = $(Throw 'A unique project name is required.')

	8< --- clipped 8< ---

	[Parameter(
		HelpMessage="`nPrefix used in the C++ Preprocessor MACRO.")]
	[ValidateScript({testPrefixParam $_})]
	[string]
	$Prefix = $(Get-FuncGuardProjectPrefix)

	8< --- clipped 8< ---

	function Get-FuncGuardProjectPrefix {
		return 	$FuncGuardSettings.("Configurations").(
			$($FuncGuardSettings.("ActiveConfiguration"))).Prefix
	}
