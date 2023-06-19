# NetCoreWinFormsWAM_Maha

# Desktop app that calls web APIs: Acquire a token by using WAM

The Microsoft Authentication Library (MSAL) calls Web Account Manager (WAM), a Windows 10+ component that acts as an authentication broker.

## WAM value proposition

Using an authentication broker such as WAM has numerous benefits:

- Enhanced security. See [Token protection](/azure/active-directory/conditional-access/concept-token-protection).
- Support for Windows Hello, conditional access, and FIDO keys.
- Integration with the Windows **Email & accounts** view.
- Fast single sign-on.
- Ability to sign in silently with the current Windows account.

## WAM limitations

- WAM is available on Windows 10 and later, and on Windows Server 2019 and later. On Mac, Linux, and earlier versions of Windows, MSAL automatically falls back to a browser.
- Azure Active Directory B2C (Azure AD B2C) and Active Directory Federation Services (AD FS) authorities aren't supported. MSAL falls back to a browser.

## WAM calling pattern

You can use the following pattern for WAM:

```csharp
    // 1. Configuration - read below about redirect URI
    var pca = PublicClientApplicationBuilder.Create("client_id")
                    .WithBroker(new BrokerOptions(BrokerOptions.OperatingSystems.Windows))
                    .Build();

    // Add a token cache; see https://learn.microsoft.com/azure/active-directory/develop/msal-net-token-cache-serialization?tabs=desktop

    // 2. Find an account for silent login

    // Is there an account in the cache?
    IAccount accountToLogin = (await pca.GetAccountsAsync()).FirstOrDefault();
    if (accountToLogin == null)
    {
        // 3. No account in the cache; try to log in with the OS account
        accountToLogin = PublicClientApplication.OperatingSystemAccount;
    }

    try
    {
        // 4. Silent authentication 
        var authResult = await pca.AcquireTokenSilent(new[] { "User.Read" }, accountToLogin)
                                    .ExecuteAsync();
    }
    // Cannot log in silently - most likely Azure AD would show a consent dialog or the user needs to re-enter credentials
    catch (MsalUiRequiredException) 
    {
        // 5. Interactive authentication
        var authResult = await pca.AcquireTokenInteractive(new[] { "User.Read" })
                                    .WithAccount(accountToLogin)
                                    // This is mandatory so that WAM is correctly parented to your app; read on for more guidance
                                    .WithParentActivityOrWindow(myWindowHandle) 
                                    .ExecuteAsync();
                                    
        // Consider allowing the user to re-authenticate with a different account, by calling AcquireTokenInteractive again                                  
    }
```

If a broker isn't present (for example, Windows 8.1, Mac, or Linux), MSAL falls back to a browser, where redirect URI rules apply.

### Redirect URI

You don't need to configure WAM redirect URIs in MSAL, but you do need to configure them in the app registration:

```
ms-appx-web://microsoft.aad.brokerplugin/{client_id}
```

### Token cache persistence

It's important to persist the MSAL token cache because MSAL continues to store ID tokens and account metadata there. For more information, see [Token cache serialization in MSAL.NET](/azure/active-directory/develop/msal-net-token-cache-serialization?tabs=desktop).

### Account for silent login

To find an account for silent login, we recommend this pattern:

- If the user previously logged in, use that account. If not, use `PublicClientApplication.OperatingSystemAccount` for the current Windows account.
- Allow the user to change to a different account by logging in interactively.

## Parent window handles

You must configure MSAL with the window that the interactive experience should be parented to, by using `WithParentActivityOrWindow` APIs.

### UI applications

For UI apps like Windows Forms (WinForms), Windows Presentation Foundation (WPF), or Windows UI Library version 3 (WinUI3), see [Retrieve a window handle](/windows/apps/develop/ui-input/retrieve-hwnd).

### Console applications

For console applications, the configuration is more involved because of the terminal window and its tabs. Use the following code:

```csharp
enum GetAncestorFlags
{   
    GetParent = 1,
    GetRoot = 2,
    /// <summary>
    /// Retrieves the owned root window by walking the chain of parent and owner windows returned by GetParent.
    /// </summary>
    GetRootOwner = 3
}

/// <summary>
/// Retrieves the handle to the ancestor of the specified window.
/// </summary>
/// <param name="hwnd">A handle to the window whose ancestor will be retrieved.
/// If this parameter is the desktop window, the function returns NULL. </param>
/// <param name="flags">The ancestor to be retrieved.</param>
/// <returns>The return value is the handle to the ancestor window.</returns>
[DllImport("user32.dll", ExactSpelling = true)]
static extern IntPtr GetAncestor(IntPtr hwnd, GetAncestorFlags flags);

[DllImport("kernel32.dll")]
static extern IntPtr GetConsoleWindow();

// This is your window handle!
public IntPtr GetConsoleOrTerminalWindow()
{
   IntPtr consoleHandle = GetConsoleWindow();
   IntPtr handle = GetAncestor(consoleHandle, GetAncestorFlags.GetRootOwner );
  
   return handle;
}
```

