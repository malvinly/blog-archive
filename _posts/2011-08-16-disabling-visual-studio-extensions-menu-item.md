---
layout: post
title: "Disabling Visual Studio extension's menu item"
date: 2011-08-16 06:01:18 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2011/08/16/disabling-visual-studio-extensions-menu-item/
---

I've created a Visual Studio extension with a new menu item that appears under the Tools menu. However, when there are no solutions open, I want to disable the menu item.

When I first create a Visual Studio package, an `Initialize` method is auto-generated:

{% raw %}
```csharp
protected override void Initialize()
{
	Trace.WriteLine (string.Format(CultureInfo.CurrentCulture, "Entering Initialize() of: {0}", this.ToString()));
	base.Initialize();

	// Add our command handlers for menu (commands must exist in the .vsct file)
	OleMenuCommandService mcs = GetService(typeof(IMenuCommandService)) as OleMenuCommandService;
	if ( null != mcs )
	{
		// Create the command for the menu item.
		CommandID menuCommandID = new CommandID(GuidList.guidMyVSPackageCmdSet, (int)PkgCmdIDList.cmdidMyCommand);
		MenuCommand menuItem = new MenuCommand(MenuItemCallback, menuCommandID );
		mcs.AddCommand( menuItem );
	}
}
```
{% endraw %}

`MenuCommand` does not have any events that I can hook into, but changing the type to `OleMenuCommand` exposes the `BeforeQueryStatus` event that allows me to execute code before the status of the command is checked.

{% raw %}
```csharp
OleMenuCommand menuItem = new OleMenuCommand(MenuItemCallback, menuCommandID );
mcs.AddCommand( menuItem );

menuItem.BeforeQueryStatus += (sender, evt) =>
{
	// I don't need this next line since this is a lambda.
	// But I just wanted to show that sender is the OleMenuCommand.
	OleMenuCommand item = (OleMenuCommand) sender;

	DTE service = (DTE) this.GetService(typeof (DTE));
	Solution solution = service.Solution;

	item.Enabled = solution.IsOpen;
};
```
{% endraw %}

However, this does not work as expected. When I first launch Visual Studio without a solution loaded, the menu item is still enabled. What I found was that the `Initialize` method is not called until the first time you click on your menu item. This code works after I've clicked on my menu item once, but I want the menu item disabled by default when Visual Studio first opens.

To disable the menu item by default, I'll need to add a command flag. In the .vsct file that is auto-generated, I can add a command flag to the button item to have it disabled by default.

{% raw %}
```xml
  <Button guid="guidMyVSPackageCmdSet" id="cmdidMyCommand" priority="0x0100" type="Button">
	<Parent guid="guidMyVSPackageCmdSet" id="MyMenuGroup" />
	<Icon guid="guidImages" id="bmpPic1" />
	<CommandFlag>DefaultDisabled</CommandFlag>
	<Strings>
	  <CommandName>cmdidMyCommand</CommandName>
	  <ButtonText>My Menu Item</ButtonText>
	</Strings>
  </Button>
```
{% endraw %}

This does successfully disable the menu item by default when Visual Studio is launched without a solution. However, since the button is now disabled, I cannot click on it, even after a solution is open. Since the package isn't loaded until the first time I click on the menu item, the package is never loaded and the `Initialize` method is never called.

To fix this problem, I'll need the package to automatically load once a solution is opened in Visual Studio, regardless if I've clicked on the menu item or not. I need to add an attribute to my package class:

{% raw %}
```csharp
[ProvideAutoLoad("f1536ef8-92ec-443c-9ed7-fdadf150da82")]
```
{% endraw %}

This attribute forces my package to load when a specific UI context is available. In this case, the GUID here represents when a solution exists. There are a number of other UI context GUIDs available under the `VSConstants` enum.

My extension now works as expected. When I launch Visual Studio without a solution open, the menu item is disabled. Once I open a solution and click on the the Tools menu, the `BeforeQueryStatus` event is triggered and my menu item is enabled.
