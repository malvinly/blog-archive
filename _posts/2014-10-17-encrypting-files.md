---
layout: post
title: "Encrypting files"
date: 2014-10-17 04:49:19 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2014/10/16/encrypting-files/
---

I have several files I need to backup to the cloud. I looked at automated backup solutions like Mozy, CrashPlan, etc..., but they were expensive compared to storage solutions like Amazon S3, Azure Storage, and others. So I figure that I can just encrypt the files myself and upload it to one of those services. I wrote the code below to encrypt my files:

{% raw %}
```csharp
void Main()
{
	string pass = "$ecret p@ssword";
	 
	string source = @"C:\file.dat";
	string encryptedDestination = @"C:\file.encrypted";
	string decryptedDestination = @"C:\file.decrypted.dat";
	
	byte[] salt = EncryptFile(pass, source, encryptedDestination);
	DecryptFile(pass, salt, encryptedDestination, decryptedDestination);
			 
	Console.WriteLine ("Done");
}

byte[] EncryptFile(string password, string source, string destination)
{	
	if (!File.Exists(source))
		throw new ArgumentException("Source does not exist.", "source");
	
	if (File.Exists(destination))
		throw new ArgumentException("Destination already exist.", "destination");
		
	byte[] IV;
	
	using (var provider = new RijndaelManaged())
	using (var passwordGen = new Rfc2898DeriveBytes(password, provider.IV))
	{
		provider.Key = passwordGen.GetBytes(provider.KeySize / 8);
		IV = provider.IV;
			
		using (var encrypt = provider.CreateEncryptor())
		using (var destinationStream = File.Create(destination))
		using (var cryptoStream = new CryptoStream(destinationStream, encrypt, CryptoStreamMode.Write))
		using (var sourceStream = new FileStream(source, FileMode.Open, FileAccess.Read, FileShare.Read))
		{
			sourceStream.CopyTo(cryptoStream);
		}
	}
	
	return IV;
}

void DecryptFile(string password, byte[] salt, string source, string destination)
{
	if (!File.Exists(source))
		throw new ArgumentException("Source does not exist.", "source");
	
	if (File.Exists(destination))
		throw new ArgumentException("Destination already exist.", "destination");
		
	using (var provider = new RijndaelManaged())
	using (var passwordGen = new Rfc2898DeriveBytes(password, salt))
	{
		provider.Key = passwordGen.GetBytes(provider.KeySize / 8);
		provider.IV = salt;
				
		using (var decrypt = provider.CreateDecryptor())
		using (var destinationStream = File.Create(destination))
		using (var cryptoStream = new CryptoStream(destinationStream, decrypt, CryptoStreamMode.Write))
		using (var sourceStream = new FileStream(source, FileMode.Open, FileAccess.Read, FileShare.Read))
		{
			sourceStream.CopyTo(cryptoStream);
		}
	}
}
```
{% endraw %}

Although the encryption code works, I still need to store the generated initialization vectors (IV) somewhere along with the encrypted files. That's kind of a hassle. Instead, I can prepend the IV to the encrypted files. When I'm decrypting files, I can read the first 16 bytes and assume that's the IV.

{% raw %}
```csharp
void EncryptFile(string password, string source, string destination)
{   
    if (!File.Exists(source))
        throw new ArgumentException("Source does not exist.", "source");
     
    if (File.Exists(destination))
        throw new ArgumentException("Destination already exist.", "destination");
     
    using (var provider = new RijndaelManaged())
    using (var passwordGen = new Rfc2898DeriveBytes(password, provider.IV))
    {
        provider.Key = passwordGen.GetBytes(provider.KeySize / 8);
                     
        using (var encrypt = provider.CreateEncryptor())
        using (var destinationStream = File.Create(destination))
		using (var cryptoStream = new CryptoStream(destinationStream, encrypt, CryptoStreamMode.Write))
		using (var sourceStream = new FileStream(source, FileMode.Open, FileAccess.Read, FileShare.Read))
		{
			destinationStream.Write(provider.IV, 0, provider.IV.Length);
			sourceStream.CopyTo(cryptoStream);
		}
    }
}
 
void DecryptFile(string password, string source, string destination)
{
    if (!File.Exists(source))
        throw new ArgumentException("Source does not exist.", "source");
     
    if (File.Exists(destination))
        throw new ArgumentException("Destination already exist.", "destination");
	
	byte[] salt = new byte[16];
	
	using (var sourceStream = new FileStream(source, FileMode.Open, FileAccess.Read, FileShare.Read))
	{
		sourceStream.Read(salt, 0, salt.Length);
	}
	
    using (var provider = new RijndaelManaged())
    using (var passwordGen = new Rfc2898DeriveBytes(password, salt))
    {
        provider.Key = passwordGen.GetBytes(provider.KeySize / 8);
        provider.IV = salt;
                 
        using (var decrypt = provider.CreateDecryptor())
        using (var destinationStream = File.Create(destination))
        using (var cryptoStream = new CryptoStream(destinationStream, decrypt, CryptoStreamMode.Write))
        using (var sourceStream = new FileStream(source, FileMode.Open, FileAccess.Read, FileShare.Read))
        {
			sourceStream.Seek(salt.Length, SeekOrigin.Begin);
            sourceStream.CopyTo(cryptoStream);
        }
    }
}
```
{% endraw %}
