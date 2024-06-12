# A basic HPC
2 min read · Dec 11, 2023

Creating a High-Performance Computing (HPC) cluster involves several technical steps, including setting up the hardware, installing and configuring the software, and ensuring network connectivity. Here’s a tutorial with command lines, focusing on a Linux-based environment. This tutorial assumes basic knowledge of Linux and networking.

## Step 1: Basic Setup
- **Install a Linux Distribution**: Choose a server-oriented distribution like CentOS, Ubuntu Server, or Debian.
    - Example: To install Ubuntu Server, download the ISO from the Ubuntu website and follow the installation instructions.
- **Update Your System**:
    ```bash
    sudo apt update && sudo apt upgrade
    ```

## Step 2: Install SSH
SSH (Secure Shell) is essential for remotely managing your cluster nodes.
- **Install SSH Server**:
    ```bash
    sudo apt install openssh-server
    ```
- **Start SSH Service**:
    ```bash
    sudo systemctl start ssh
    ```
- **Enable SSH on Boot**:
    ```bash
    sudo systemctl enable ssh
    ```

## Step 3: Setting Up a Network
Configure a private network for your cluster nodes. This setup assumes you have a dedicated network interface for the cluster.
- **Edit Network Configuration**:
    ```bash
    sudo nano /etc/network/interfaces
    ```
    Add lines similar to the following, replacing `eth1` and IP addresses with your specific details:
    ```plaintext
    auto eth1
    iface eth1 inet static
        address 192.168.1.1
        netmask 255.255.255.0
    ```
- **Restart Network Service**:
    ```bash
    sudo systemctl restart networking
    ```

## Step 4: Install and Configure NFS
Network File System (NFS) allows you to share directories among various nodes in the cluster.
- **Install NFS Server**:
    ```bash
    sudo apt install nfs-kernel-server
    ```
- **Configure Exports**:
    ```bash
    sudo nano /etc/exports
    ```
    Add a line like:
    ```plaintext
    /home 192.168.1.0/24(rw,sync,no_subtree_check)
    ```
- **Export the Shared Directories**:
    ```bash
    sudo exportfs -a
    ```
- **Start NFS Service**:
    ```bash
    sudo systemctl start nfs-kernel-server
    ```

## Step 5: Setting Up Compute Nodes
Repeat Steps 1 to 3 for each compute node, ensuring each node has a unique IP address but is on the same subnet.

## Step 6: Install MPI
MPI (Message Passing Interface) is crucial for parallel computing.
- **Install MPI**:
    ```bash
    sudo apt install mpich
    ```

## Step 7: Cluster Management and Job Scheduling
Install and configure a job scheduler like Slurm or TORQUE. Here’s a basic setup for Slurm:
- **Install Slurm**:
    ```bash
    sudo apt install slurm-wlm
    ```
- **Configure Slurm**:
    ```bash
    sudo nano /etc/slurm-llnl/slurm.conf
    ```
    Configure the control and compute nodes according to your cluster’s setup.
- **Start Slurm Services**:
    ```bash
    sudo systemctl start slurmctld
    sudo systemctl start slurmd
    ```

## Step 8: Testing the Cluster
- **Run a Test MPI Job**:
    Create a simple MPI test program and compile it:
    ```bash
    mpicc -o mpi_test mpi_test.c
    sbatch mpi_test
    ```

## Conclusion
This tutorial outlines the fundamental steps for setting up a basic HPC cluster. Each HPC setup will have unique requirements, and further customization might be needed. For more detailed configurations, especially for larger or more specialized clusters, consult the documentation for each software package and consider seeking advice from HPC experts.

Tags: Hpc, Networking, Linux, Tutorial

# Building a Cryptographic Key Management System in .NET Core

In this tutorial, we will walk through the process of building a cryptographic key management system (KMS) using .NET Core. This KMS will support both RSA and AES encryption algorithms and provide various key management functionalities such as creating, activating, deactivating, revoking, and destroying keys.

## Introduction to RSA and AES Cryptography

**RSA** (Rivest-Shamir-Adleman) is a widely used public-key cryptographic algorithm that relies on the mathematical properties of large prime numbers. It is often used for secure data transmission.

**AES** (Advanced Encryption Standard) is a symmetric key encryption standard widely used across the globe. It encrypts data in fixed-size blocks and is known for its speed and security.

## Project Structure

The project is divided into several key files, each serving a distinct purpose. Let's break down each file and its role within the project.

### Program.cs

This is the entry point of our application.

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

The `Program` class contains the `Main` method, which is the starting point of the application. It configures and runs the web host using the `Startup` class.

### Startup.cs

This class configures services and the app's request pipeline.

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        services.AddSingleton<CryptographyProviderFactory>();
        services.AddSingleton<KeyStoreManager>();
        services.AddTransient<AESProvider>();
        services.AddTransient<RSAProvider>();

        services.AddSwaggerGen(c =>
        {
            c.SwaggerDoc("v1", new OpenApiInfo
            {
                Title = "KMS API",
                Version = "v1",
                Description = "A simple example ASP.NET Core Web API for KMS",
                Contact = new OpenApiContact
                {
                    Name = "Hamza ELJAOUHARI",
                    Email = "hamza.eljaouhari.1995@gmail.com"
                }
            });
        });

        services.AddCors(options =>
        {
            options.AddPolicy("AllowAll",
                builder => builder
                    .AllowAnyOrigin()
                    .AllowAnyMethod()
                    .AllowAnyHeader());
        });
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseHttpsRedirection();
        app.UseRouting();
        app.UseCors("AllowAll");
        app.UseAuthorization();
        app.UseSwagger();
        app.UseSwaggerUI(c =>
        {
            c.SwaggerEndpoint("/swagger/v1/swagger.json", "KMS API V1");
            c.RoutePrefix = string.Empty;
        });
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

- **ConfigureServices**: Adds services to the container. Registers `CryptographyProviderFactory` and `KeyStoreManager` as singletons and sets up Swagger for API documentation.
- **Configure**: Sets up the request pipeline, enabling HTTPS redirection, routing, CORS, and Swagger UI.

### CryptographyController.cs

This controller handles API requests related to cryptographic operations.

```csharp
[ApiController]
[Route("api/[controller]")]
public class CryptographyController : ControllerBase
{
    private readonly CryptographyProviderFactory _factory;
    private readonly KeyStoreManager _keyStoreManager;

    public CryptographyController(CryptographyProviderFactory factory, KeyStoreManager keyStoreManager)
    {
        _factory = factory;
        _keyStoreManager = keyStoreManager;
    }

    [HttpGet("list")]
    public IActionResult GetAllKeys()
    {
        var keys = _keyStoreManager.GetAllKeys();
        return Ok(keys);
    }

    [HttpPost("create")]
    public IActionResult CreateKey([FromBody] CreateKeyRequest request)
    {
        try
        {
            var provider = _factory.GetCryptographyProvider(request.Algorithm);
            var keyId = provider.CreateKey(request.Algorithm, request.KeySize);
            return Ok(keyId);
        }
        catch (Exception ex)
        {
            return BadRequest(ex.Message);
        }
    }

    [HttpPost("encrypt")]
    public IActionResult Encrypt([FromBody] EncryptRequest request)
    {
        try
        {
            var provider = _factory.GetCryptographyProvider(request.Algorithm);
            var encryptedData = provider.Encrypt(request.Data, request.KeyId);
            return Ok(encryptedData);
        }
        catch (Exception ex)
        {
            return BadRequest(ex.Message);
        }
    }

    [HttpPost("decrypt")]
    public IActionResult Decrypt([FromBody] DecryptRequest request)
    {
        try
        {
            var provider = _factory.GetCryptographyProvider(request.Algorithm);
            var decryptedData = provider.Decrypt(request.Data, request.KeyId);
            return Ok(decryptedData);
        }
        catch (Exception ex)
        {
            return BadRequest(ex.Message);
        }
    }

    // Other key lifecycle methods (Activate, Deactivate, Destroy, Revoke, Archive, Recover, GetKeyInfo) follow the same pattern.
}
```

This controller defines endpoints for:
- Listing all keys.
- Creating a new key.
- Encrypting data.
- Decrypting data.
- Managing key lifecycle (activate, deactivate, destroy, etc.).

### CryptographyProviderFactory.cs

This factory class creates instances of cryptographic providers based on the requested algorithm.

```csharp
public class CryptographyProviderFactory
{
    private readonly IServiceProvider _serviceProvider;

    public CryptographyProviderFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IKeyLifeCycleManager GetCryptographyProvider(string algorithm)
    {
        switch (algorithm.ToUpper())
        {
            case "AES":
                return _serviceProvider.GetRequiredService<AESProvider>();
            case "RSA":
                return _serviceProvider.GetRequiredService<RSAProvider>();
            default:
                throw new ArgumentException("Unsupported algorithm.");
        }
    }
}
```

The factory pattern is used to create instances of cryptographic providers (`AESProvider`, `RSAProvider`) based on the algorithm specified. This promotes loose coupling and adheres to the Open/Closed Principle.

### KeyStoreManager.cs

Manages key storage and lifecycle.

```csharp
public class KeyStoreManager
{
    private Dictionary<string, KeyInfo> KeyStore = new Dictionary<string, KeyInfo>();
    private const string LogFilePath = "key_usage_log.txt";

    public void AddKey(string keyId, byte[] keyData, string algorithm)
    {
        KeyStore[keyId] = new KeyInfo
        {
            KeyId = keyId,
            KeyData = keyData,
            Status = "Active",
            AlgorithmType = algorithm
        };
    }

    public KeyInfo GetKeyInfo(string keyId)
    {
        ValidateKeyExists(keyId);
        return KeyStore[keyId];
    }

    public IEnumerable<KeyInfo> GetAllKeys()
    {
        return KeyStore.Values;
    }

    // Other methods: ActivateKey, DeactivateKey, DestroyKey, RevokeKey, ArchiveKey, RecoverKey

    public void LogKeyUsage(string keyId, string operation)
    {
        File.AppendAllText(LogFilePath, $"{DateTime.Now}: KeyID {keyId} used for {operation}\n");
    }

    public void ValidateKeyExists(string keyId)
    {
        if (!KeyStore.ContainsKey(keyId))
        {
            throw new KeyNotFoundException($"Key with ID {keyId} does not exist.");
        }
    }
}
```

The `KeyStoreManager` handles the storage and lifecycle of cryptographic keys. It provides methods to add, retrieve, and manage keys, including logging key usage for auditing purposes.

### AESProvider.cs and RSAProvider.cs

These classes implement cryptographic operations specific to AES and RSA algorithms.

```csharp
public class AESProvider : ProviderBase
{
    public AESProvider(KeyStoreManager keyStoreManager) : base(keyStoreManager) { }

    public override string CreateKey(string keyType, int keySize)
    {
        if (keyType != "AES") throw new ArgumentException("Unsupported key type for AESProvider.");

        using (var aes = Aes.Create())
        {
            aes.KeySize = keySize;
            aes.GenerateKey();
            string keyId = Guid.NewGuid().ToString();
            keyStoreManager.AddKey(keyId, aes.Key, "AES");
            return keyId;
        }
    }

    public override string Encrypt(string text, string keyId)
    {
        var keyInfo = keyStoreManager.GetKeyInfo(keyId);
        using (var aes = Aes.Create())
        {
            aes.Key = keyInfo.KeyData;
            aes.GenerateIV();
            var encryptor = aes.CreateEncryptor(aes.Key, aes.IV);
            using (var ms = new MemoryStream())
            {
                ms.Write(aes.IV, 0, aes.IV.Length); // Prepend the IV
                using (var cs = new CryptoStream(ms, encryptor, CryptoStreamMode.Write))
                {
                    byte[] dataBytes = Encoding.UTF8.GetBytes(text);
                    cs.Write(dataBytes, 0, dataBytes.Length);
                }
                return Convert.ToBase64String(ms.ToArray());
            }
        }
   

 }

    public override string Decrypt(string encryptedText, string keyId)
    {
        var keyInfo = keyStoreManager.GetKeyInfo(keyId);
        using (var aes = Aes.Create())
        {
            aes.Key = keyInfo.KeyData;
            byte[] iv = new byte[aes.BlockSize / 8];
            byte[] encryptedBytes = Convert.FromBase64String(encryptedText);
            Array.Copy(encryptedBytes, iv, iv.Length);
            aes.IV = iv;

            var decryptor = aes.CreateDecryptor(aes.Key, aes.IV);
            using (var ms = new MemoryStream())
            {
                using (var cs = new CryptoStream(ms, decryptor, CryptoStreamMode.Write))
                {
                    cs.Write(encryptedBytes, iv.Length, encryptedBytes.Length - iv.Length);
                }
                byte[] decryptedBytes = ms.ToArray();
                return Encoding.UTF8.GetString(decryptedBytes);
            }
        }
    }
}

public class RSAProvider : ProviderBase
{
    public RSAProvider(KeyStoreManager keyStoreManager) : base(keyStoreManager) { }

    public override string CreateKey(string keyType, int keySize)
    {
        if (keyType != "RSA") throw new ArgumentException("Unsupported key type for RSAProvider.");

        using (var rsa = RSA.Create(keySize))
        {
            string keyId = Guid.NewGuid().ToString();
            RSAParameters parameters = rsa.ExportParameters(true);
            byte[] serializedParameters = SerializeRsaParameters(parameters);
            keyStoreManager.AddKey(keyId, serializedParameters, "RSA");
            return keyId;
        }
    }

    public override string Encrypt(string text, string keyId)
    {
        var keyInfo = keyStoreManager.GetKeyInfo(keyId);
        using (var rsa = RSA.Create())
        {
            rsa.ImportParameters(DeserializeRsaParameters(keyInfo.KeyData));
            byte[] dataBytes = Encoding.UTF8.GetBytes(text);
            byte[] encryptedBytes = rsa.Encrypt(dataBytes, RSAEncryptionPadding.OaepSHA256);
            return Convert.ToBase64String(encryptedBytes);
        }
    }

    public override string Decrypt(string encryptedText, string keyId)
    {
        var keyInfo = keyStoreManager.GetKeyInfo(keyId);
        using (var rsa = RSA.Create())
        {
            rsa.ImportParameters(DeserializeRsaParameters(keyInfo.KeyData));
            byte[] encryptedBytes = Convert.FromBase64String(encryptedText);
            byte[] decryptedBytes = rsa.Decrypt(encryptedBytes, RSAEncryptionPadding.OaepSHA256);
            return Encoding.UTF8.GetString(decryptedBytes);
        }
    }

    private byte[] SerializeRsaParameters(RSAParameters parameters)
    {
        using (var ms = new MemoryStream())
        {
            using (var writer = new BinaryWriter(ms))
            {
                WriteIfNotNull(writer, parameters.Modulus);
                WriteIfNotNull(writer, parameters.Exponent);
                WriteIfNotNull(writer, parameters.P);
                WriteIfNotNull(writer, parameters.Q);
                WriteIfNotNull(writer, parameters.DP);
                WriteIfNotNull(writer, parameters.DQ);
                WriteIfNotNull(writer, parameters.InverseQ);
                WriteIfNotNull(writer, parameters.D);
            }
            return ms.ToArray();
        }
    }

    private void WriteIfNotNull(BinaryWriter writer, byte[] data)
    {
        if (data != null)
        {
            writer.Write(data.Length);
            writer.Write(data);
        }
        else
        {
            writer.Write(0);
        }
    }

    private RSAParameters DeserializeRsaParameters(byte[] data)
    {
        using (var ms = new MemoryStream(data))
        {
            using (var reader = new BinaryReader(ms))
            {
                RSAParameters parameters = new RSAParameters
                {
                    Modulus = ReadNextArray(reader),
                    Exponent = ReadNextArray(reader),
                    P = ReadNextArray(reader),
                    Q = ReadNextArray(reader),
                    DP = ReadNextArray(reader),
                    DQ = ReadNextArray(reader),
                    InverseQ = ReadNextArray(reader),
                    D = ReadNextArray(reader)
                };
                return parameters;
            }
        }
    }

    private byte[] ReadNextArray(BinaryReader reader)
    {
        int length = reader.ReadInt32();
        return length > 0 ? reader.ReadBytes(length) : null;
    }
}
```

These provider classes handle the specific operations for their respective cryptographic algorithms:
- **AESProvider**: Creates AES keys, encrypts, and decrypts data using AES.
- **RSAProvider**: Creates RSA keys, encrypts, and decrypts data using RSA.

### Key Lifecycle Management

Each key lifecycle method (Activate, Deactivate, Destroy, etc.) in the `CryptographyController` and `KeyStoreManager` is crucial for maintaining the integrity and security of cryptographic keys. Here’s a brief overview:

- **ActivateKey**: Makes a key available for use.
- **DeactivateKey**: Temporarily disables a key.
- **DestroyKey**: Permanently deletes a key.
- **RevokeKey**: Marks a key as compromised.
- **ArchiveKey**: Moves a key to long-term storage.
- **RecoverKey**: Restores an archived key to active use.

These operations ensure that keys are managed securely throughout their lifecycle, helping to prevent unauthorized access and usage.

## Conclusion

This tutorial provides a comprehensive overview of building a cryptographic key management system in .NET Core. By following these steps, you can create a robust system for managing cryptographic keys, ensuring secure encryption and decryption of data.

For a deeper dive into the code and to explore additional features, check out the full repository on GitHub: [KMS .NET Core Repository](https://github.com/hamza-eljaouhari/klms-net-core).

# Hi there 👋

Welcome to my GitHub profile! I'm Hamza Eljaouhari, a passionate developer with a love for electric guitar, basketball, coding, and movies. I've been playing the electric guitar for about 15 years and enjoy combining my interests in technology and music.

🔭 I’m currently working on:

## Cryptographical Keys Lifecycle Management Application

This project is designed to handle the lifecycle of cryptographic keys for both AES and RSA encryption. It includes functionalities for creating, activating, deactivating, archiving, recovering, revoking, and destroying keys. The project supports AES with key sizes of 128, 192, and 256 bits, and RSA with key sizes of 1024, 2048, 3072, and 4096 bits.

- [React GUI](https://github.com/hamza-eljaouhari/kms-react-gui)
- [.NET Core API](https://github.com/hamza-eljaouhari/klms-net-core)

## Interactive Fretboard and Circle of Fifths

The Interactive Fretboard and Circle of Fifths project is designed to assist musicians, particularly guitarists, in improving their skills, composing music, and learning new songs on the guitar. By combining a dynamic fretboard visualization with the Circle of Fifths, this tool provides an intuitive interface for exploring musical concepts and practicing techniques.

## Healer - Food Supplements Suggester

Healer is a Food Supplements Suggester designed to assist users in improving their nutritional intake and addressing specific health concerns through the recommendation of suitable food supplements. By providing functionalities such as nutritional medical results analysis and symptom evaluation, Healer offers personalized recommendations tailored to individual needs.

- [Healer](https://github.com/hamza-eljaouhari/healer)

## Prayerful - Prayer Generator

Prayerful is a prayer generator designed to provide users with personalized prayers. This project aims to offer spiritual support by generating prayers based on user input and preferences. It includes an Express.js backend and a React frontend, with a Next.js version in development.

- [Express.js App](https://github.com/hamza-eljaouhari/express-prayerful)
- [Client React.js App](https://github.com/hamza-eljaouhari/prayerful)
- [Next.js Redo](https://github.com/hamza-eljaouhari/next-prayers)

🌱 I’m currently learning:
- Advanced optimization concepts and techniques
- New ways to approach difficult companies' business problems
- Music theory and composition

👯 I’m looking to collaborate on:
- Cryptography projects
- Music-related software
- Banking applications
- Security applications

🤔 I’m looking for help with:
- Implementing advanced security features
- Enhancing user experience for my applications
- Exploring new frameworks and technologies

💬 Ask me about:
- Guitar techniques and music theory
- Full-stack development with C# / .NET Core and React

📫 How to reach me:
- Email: [hamza.eljaouhari.1995@gmail.com](mailto:hamza.eljaouhari.1995@gmail.com)

😄 Pronouns:
- He/Him

⚡ Fun fact:
- I love blending my technical skills with my architecturing capacities to create innovative tools and applications.

---

Thank you for visiting my profile! Feel free to explore my repositories and reach out if you'd like to connect or collaborate on exciting projects.
