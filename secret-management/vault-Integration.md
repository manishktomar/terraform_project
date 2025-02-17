# HashiCorp Vault Setup

### 1. Install Vault on the EC2 instance
- To install Vault on the EC2 instance, you can use the following steps
    ```
    sudo apt update && sudo apt install gpg
    wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
    gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
    sudo apt update
    ```

- Install Vault
    ```
    sudo apt install vault
    ```

### 2. Start Vault
- To start Vault, you can use the following command:
    ```
    vault server -dev -dev-listen-address="0.0.0.0:8200"
    ```
  ![image](https://github.com/user-attachments/assets/7999c90a-32f8-45ee-9075-bfc32c719737)

- Next, authenticate using the root token generated by Vault:
    ```
    export VAULT_ADDR='http://0.0.0.0:8200'
    vault login <root-token>
    ```
- **Enable 8200 port on EC2 Machine** 
    ```
    EC2 Instance > Security > Security Group
    Edit Inbound Rules > Add Rule
    Type = Custom TCP | Port = 8200
    Save Rules
    ```

### 3. Login on Vault
- Use your EC2 Public IP with port 8200 for Login Vault
    ```
    0.0.0.0:8200
    ```
    ![image](https://github.com/user-attachments/assets/40f82586-9b42-4fd9-bb60-f1020763a0d7)
    ![image](https://github.com/user-attachments/assets/be14703a-a6bf-41d7-96f0-d024fd9e483e)

- Choose Token > Enter Root Token

### 4. Setup on Vault
Once login, choose the KV or any other as per the requirement
     ![image](https://github.com/user-attachments/assets/c5e0168a-b768-4612-accf-024afa1097a0)

    ```
    Choose = KV (KeyValue)
    Path = **kv** > Enable Engine
    Path for this secret = **test-vault**
    Secret Data = username (key) | xxxxx (password)
    ```
    
### 5. Configure Terraform to read the secret from Vault.
- Enable AppRole Authentication:
    ```
    vault auth enable approle
    ```

- Create an policies:
    ```
    vault policy write **terraform** - <<EOF
    path "*" {
    capabilities = ["list", "read"]
    }
    ```
  
    ```
    path "secrets/data/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
    }

    path "kv/data/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
    }


    path "secret/data/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
    }

    path "auth/token/create" {
    capabilities = ["create", "read", "update", "list"]
    }
    EOF
    ```

- Create the AppRole
Now you'll need to create an AppRole with appropriate policies and configure its authentication settings.
    ```
    vault write auth/approle/role/terraform \
    secret_id_ttl=10m \
    token_num_uses=10 \
    token_ttl=20m \
    token_max_ttl=30m \
    secret_id_num_uses=40 \
    token_policies=terraform
    ```

### 6. Generate Role ID and Secret ID:
After creating the AppRole, you need to generate a Role ID and Secret ID pair. The Role ID is a static identifier, while the Secret ID is a dynamic credential.

- Generate Role ID:
    ```
    vault read auth/approle/role/terraform/role-id
    ```

- Generate Secret ID:
    ```
    vault write -f auth/approle/role/terraform/secret-id
    ```

### 7. Add Role ID and Secret ID in main.tf:
- Add Valut details in main.tf and test
    ```
    provider "aws" {
      region = "us-east-1"
    }
    
    provider "vault" {
      address = "http://AWS-Public-IP:8200"
      skip_child_token = true
    
      auth_login {
        path = "auth/approle/login"
    
        parameters = {
          role_id = "<>"
          secret_id = "<>"
        }
      }
    }
    
    data "vault_kv_secret_v2" "example" {
      mount = "kv" // change it according to your mount
      name  = "test-vault" // change it according to your secret
    }
    ```
- Apply Terraform CMD
    ```
    terraform init
    terraform plan
    terraform validate
    terrafrom apply
    ```
