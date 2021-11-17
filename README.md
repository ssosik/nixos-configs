# nixos-configs

Following this doc: https://www.vultr.com/docs/how-to-install-nixos-on-a-vultr-vps

Created custom ISO image from https://channels.nixos.org/nixos-21.05/latest-nixos-minimal-x86_64-linux.iso

## Create an SSH key, only do this if you don't want to reuse an existing key

```
ssh-keygen -t ed25519 -b 4096 -f .ssh/sosik -C '10/29/2021'
<password>
# Add the key
ssh-add .ssh/sosik
# Get the key fingerprint
ssh-add -L
```

## Temporarily enable SSH

On the Vultr VM web console <Send Clipboard>
```
mkdir .ssh
curl https://raw.githubusercontent.com/ssosik/nixos-configs/main/ssh.keys > ~/.ssh/authorized_keys
```

Then in iTerm with the SSH key loaded: `ssh nixos@8.9.11.190`

## Su root
sudo -s

## Partition /dev/vda

Documentation: https://www.vultr.com/docs/how-to-install-nixos-on-a-vultr-vps#Create_the_File_System
GNU Parted Reference https://www.gnu.org/software/parted/manual/html_node/Running-Parted.html

Partition, format, and mount disk

Since I'm running with less than 4GB of RAM create a swap
```
parted /dev/vda -s 'mklabel msdos mkpart primary 1MiB -1GiB mkpart primary linux-swap -1GiB 100%' && \
    parted -l && \
    mkfs.ext4 -L root /dev/vda1 && \
    mkswap -L swap /dev/vda2 && \
    swapon /dev/vda2 && \
    mount /dev/disk/by-label/root /mnt
```

## Configure NixOS

```
nixos-generate-config --root /mnt

# Set hostname
sed -i 's/  # networking.hostName = "nixos";.*/  networking.hostName = "testmachine";/' /mnt/etc/nixos/configuration.nix
# grub device
sed -i 's|  # boot.loader.grub.device = "/dev/sda";.*|  boot.loader.grub.device = "/dev/vda";|' /mnt/etc/nixos/configuration.nix
# enable SSH
sed -i 's|  # services.openssh.enable = true;|  services.openssh.enable = true;\n  services.openssh.permitRootLogin = "no";\n  services.openssh.passwordAuthentication = false;|' /mnt/etc/nixos/configuration.nix
# Add users
sed -i 's|      ./hardware-configuration.nix|      ./hardware-configuration.nix\n      ./users.nix|' /mnt/etc/nixos/configuration.nix

cat <<EOF > /mnt/etc/nixos/users.nix
{ config, lib, pkgs, modulesPath, ... }:

{
  users.mutableUsers = false;
  
  # Add a user.
  users.users.nixos = {
    isNormalUser = true;
  
    # Add a hashed password, overrides initialPassword.
    # See below. NOTE: escape dollar signs here if running this as a bash command
    hashedPassword = "\$6\$1OvnB1HeKH\$9/exwQNwcCUknibmcK2i745uEO/7nJe/53aPAyyvFaadM3zgxSuWcMnQ8NpZZGQegUz2dC5JXgSGk1oCZcjWn.";
  
    extraGroups = [ "wheel" ];
    openssh.authorizedKeys.keys = [
      # Authorize the SSH public key from 'key.pub'.
      # Remove this statement if you use password
      # authentication.
      (builtins.readFile ./key.pub)
    ];
  };
}
EOF

cp /home/nixos/.ssh/authorized_keys /mnt/etc/nixos/key.pub

```

## Enable and use Flakes

```
nix-shell -p nixUnstable git

cat <<EOF > /mnt/etc/nixos/flake.nix
{
  description = "system configuration flake";

  inputs = {
    # Replace this with any nixpkgs revision you want to use.
    # See a list of potential revisions at
    # https://github.com/NixOS/nixpkgs/branches/active
    nixpkgs.url = "nixpkgs/release-21.05";
  };

  outputs = inputs@{ self, nixpkgs }:
    let
      lib = nixpkgs.lib;
      nixConf = pkgs: {
        environment.systemPackages = [ pkgs.git ];
        nix = {
          package = pkgs.nixFlakes;
          extraOptions = ''
            experimental-features = nix-command flakes
          '';
          autoOptimiseStore = true;
          gc = {
            automatic = true;
            dates = "weekly";
          };
        };
      };
    in
    {
      # Replace machineName with your desired hostname.
      nixosConfigurations.testmachine = nixpkgs.lib.nixosSystem rec {
        system = "x86_64-linux";

        modules = [
          (nixConf nixpkgs.legacyPackages.${system})
          ./configuration.nix
        ];
      };
    };
}
EOF

cd /mnt/etc/nixos
git init .
git branch -m main
git add key.pub *.nix
```

## Install

https://www.vultr.com/docs/how-to-install-nixos-on-a-vultr-vps#Install

```
nixos-install --no-root-passwd --flake /mnt/etc/nixos#testmachine
```

Reboot and verify
- Go back to the dashboard
- Select your server.
- Click the Settings tab.
- Find the Custom ISO sidebar in the tab.
- Remove the ISO. This will reboot the instance.
- SSH into the machine after it boots.
    - Need to remove previous entry from .ssh/known_hosts on the local machine
