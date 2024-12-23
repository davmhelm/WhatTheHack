# bgp.sh
Improvements

### Todo
- [ ] Better SSH private/public key pair handling, ref https://learn.microsoft.com/en-us/azure/virtual-machines/ssh-keys-azure-cli
     - [ ] Generate SSH public/private keypair, store public key in Azure SSH key resource

        ```bash
        # Borrowing from https://github.com/Azure/azure-cli/blob/dev/src/azure-cli/azure/cli/command_modules/vm/manual/custom.py#L18
        sshpath=~/.ssh
        sshfile=id_ed25519_$(date +%s_%N)

        ssh-keygen -t ed25519 -f "$sshpath/$sshfile" -C "$default_username" -N "" -q
        az sshkey create -n keyname -l $location -g $rgname \
           --encryption-type ed25519 --public-key "@$sshpath/$sshfile.pub"
        ```
        - how to get filename of SSH Private key for the next part?
    - [ ] Store private key in Azure Key Vault secret
        - Create Key Vault resource
        - Need to base64 encode the SSH private key so formatting isn't lost when storing, ref https://kritidynamics.substack.com/p/how-to-safely-store-and-retrieve
        - Use base64 decode after retrieving to unpack secret

        ```bash
        # Borrowing from https://learn.microsoft.com/en-us/azure/key-vault/secrets/quick-create-cli
        vaultname=kv-$(uuidgen | tr -d "-" | head -c 21)
        az keyvault check-name -n "$vaultname"
        vaultid=$(az keyvault create -n "$vaultname" -l $location -g $rgname --sku standard --query "id" -o tsv)
        az role assignment create --role "Key Vault Administrator" --scope $vaultid --assignee "$(az account show --query 'user.name' -o tsv)"
        az keyvault secret set --vault-name "$vaultname" --name "SSH_PrivateKey" -f $sshpath/$sshfile -e base64
        ```

- [ ] Deploy Bastion, ref https://learn.microsoft.com/en-us/azure/bastion/create-host-cli
    - Separate VNet, separate address space from sites (something like 192.168.255.0/24 should suffice), rely on VNet peering to lab VNets so only one Bastion is needed
    - Bastion resource (Standard SKU) + Public IP
        - Features required: enable-ip-connect, enable-tunneling, see https://learn.microsoft.com/en-us/cli/azure/network/bastion?view=azure-cli-latest#az-network-bastion-create
    - Order of operations: CSR configurations via SSH will depend on this.
        - Create Bastion first and wait for availability? Likely.
        - When to do VNet peering? Probably at time of lab VNet creations.

- [ ] Replace use of SSH with Azure Bastion SSH, ref https://learn.microsoft.com/en-us/azure/bastion/connect-vm-native-client-linux#ssh
    - [ ] Improve readability of SSH command options by defining once and re-using? See https://stackoverflow.com/a/56960067