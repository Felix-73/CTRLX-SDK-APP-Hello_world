# Generate Package Manifest during Installation

## Avant propo

Ce mimi projet provient du sdk du Ctrlx [samples-snap/generate-manifest](https://github.com/boschrexroth/ctrlx-automation-sdk/tree/main/samples-snap/generate-manifest) je l'ai légerement simplifié en enlevant la fonctionnalité wamerican qui permet de generer des nom aléatoires et ajouté des commentaires. Il permet de comprendre la structure des applications snap et son intégration dans l'environnement ctlrX.

## Goal

For a few use cases, it might be necessary to generate a package-manifest depending on the ctrlX CORE or on the setup in is installed. This is possible during installation by using snap hook mechanism. This how-to describes how this can be resolved.

The snap build in this examples takes two random words and uses them to generate a menu entry in the ctrlX CORE sidebar. It will look like this:

## Precondition

Basic understanding of:

- [snap interfaces](https://snapcraft.io/docs/interface-management)
- [hooks](https://snapcraft.io/docs/supported-snap-hooks)
- [package-manifest](https://boschrexroth.github.io/ctrlx-automation-sdk/package-manifest.html)

## Les parties suivantes ont déjà été réalisées mais peuvent servir pour comprendre l'architecture d'une app snap. Pour build l'app, aller à la dernière partie.
### Creating the snap 

Choose an empty folder and initialize the snap enviroment by using snapcraft

``` bash
snapcraft init
```

Create the folders as in the following image:

```ls
dump/
  bin/
  package-assets/
snap/
  hooks/
  snapcraft.yaml
.gitignore
```

The folder "dump" contains files that will be copied into our snap. This includes scripts inside the "bin" folder and the package-manifest.template used to blueprint when generating the actual package-manifest. Next to the snapcraft.yaml we need the hooks folder where we will later add the "hook" scripts.

After that open the snapcraft.yaml file and edit it as following


```yaml
name: changing-world
base: core20
version: '1.0.0'
summary: Simple snap with random menu entry
description: |
  This is a sample snap that generates a random menu entry when getting installed or updated
 
grade: stable
confinement: strict
 
parts:
  dump:
    plugin: dump
    source: ./dump
    stage-packages:
      - jq
apps:
  my-service:
    command: bin/service.sh
    daemon: simple
 
slots:
  package-assets:
    interface: content
    content: package-assets
    source:
      read:
      - $SNAP_DATA/package-assets/$SNAPCRAFT_PROJECT_NAME
```

The dump part is used to copy the files from the dump folder into the snap and to install two packages used to generate the package-manifest. The "wamerican" package is a dictionary of american english words, the jq package provides the jq tool to manipulate json files using shell.

The slot "package-assets" is similar to the one described in the SDK, the only difference is that it reference $SNAP_DATA instead of $SNAP, to provide a writeable directory.

We added here a my-service app, this is just a simple daemon which logs a string to stdout for demo purpose, see


```bash
#!/bin/bash
 
while true
do
    echo "Hello changing world"
    sleep 10
done
```

### The script

We will now add the script in the "dump/bin" folder by creating a file called "generate_manifest.sh" and make it executable (chmod +x)

```ls
dump/
  bin/
    generate_manifest.sh
```

Here you will find the content


```bash
#!/bin/bash -x
 
# Update json
NAME="toto"
mkdir -p $SNAP_DATA/package-assets/$SNAP_NAME
$SNAP/usr/bin/jq ".menus.sidebar[].title = \"$NAME\"" \
    $SNAP/package-assets/changing-world.package-manifest.json.template > $SNAP_DATA/package-assets/changing-world/changing-world.package-manifest.json 
```

In line 4 we give a name

In line 5 we prepare the directory to ensure it exists.

In line 6-7 we use jq to change the existing menus.sidebar.title of the template to "NAME" and write it into the package-assets folder.

To make this work we need to create the script template in the "dump/package-assets" folder:

[changing-world.package-manifest.json.template](dump/package-assets/changing-world.package-manifest.json.template)

```json
{
  "$schema": "https://json-schema.boschrexroth.com/ctrlx-automation/ctrlx-core/apps/package-manifest/package-manifest.v1.3.schema.json",
  "version": "1.0.0",
  "id": "changing-world",
  "menus": {
    "sidebar": [
      {
        "id": "changing-world",
        "title": "",
        "icon": "bosch-ic-automation",
        "link": "0"
      }
    ]
  }
}
```

As you can see, the title property is empty.

### The hooks

Now we need to execute the script to generate the package-manifest on the installation and every update of the snap. So we need to add the corresponding hooks:

```ls
snap/
  hooks/
    install
    post-refresh
```

Both need to be executable (chmod +x). Both have the same content as below:


```bash
#!/bin/bash
 
# Generate manifest
$SNAP/snap/command-chain/snapcraft-runner  generate_manifest.sh
```

It executes the generate_manifest.sh mentioned above using the snaps environment by using the snapcraft-runner, it sets all required environment variables (e.g. PATH). It is generated by the snap itself if you have defined an app in your snap.

### Interface definitions

Interfaces (slots and plugs) can be declared in two different ways:

- Specific for each app and hook (Preferred solution)
- Globally to be valid for all defined apps and hooks
#### Examples

Specific declaration:

Each app and each hook defines all needed interfaces.

```yaml
apps:
  example:
    command: bin/sh
    plugs:
      wayland:
      x11:
  example2:
    command: bin/sh2
    plugs:
      wayland:
      x11:
hooks:
  configure:
    plugs:
      wayland:
      x11:
```

Globally defined:

The plugs wayland and x11 are valid for the apps example, example2 as well as the configure-hook.

```yaml
apps:
  example:
    command: bin/sh
  example2:
    command: bin/sh2

hooks:
  configure:

plugs:
  wayland:
  x11:
```



Both definitions lead to the same result.

Nevertheless if you mix these two, this could lead to unexpected behaviour.

Bad Example:

In the following yaml, the global plugs wayland and x11 can only be accessed by the example app. The app example2 and the configure hook have no interfaces declared, which is probably not the desired result.

```yaml
apps:
  example:
    command: bin/sh
    plugs:
      wayland:
      x11:
      opengl:
  example2:
    command: bin/sh2
hooks:
  configure:
plugs:
  wayland:
  x11:
```

Therefore declaring all interfaces specific for each app and hook should be the preferred solution. If your snapcraft.yaml does not contain any apps or hooks at all, then declaring interfaces globally is the right approach. 


## Pour snapper = > Build and run 
```./build-snap-amd64.snap```

```./build-snap-armd64.snap```

Penser à donner les permissions avec ```chmod ```

install it on your ctrlX CORE.

Thats it.
