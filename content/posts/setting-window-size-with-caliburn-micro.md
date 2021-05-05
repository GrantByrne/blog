---
title: "Setting Window Size With Caliburn Micro"
date: 2021-05-04T22:05:27-04:00
tags: [csharp, wpf, caliburn-micro]
---

This is something that has actually bugged me for a while. Once I figured it out, it annoyed me that I didn't figure it out sooner.

When displaying a window in Caliburn.Micro, you can set attributes about the Window object when calling it.

**So, let's say you want to set the height and width on the window to 600 x 300:**

First, you would start with something like this:

{{< highlight csharp >}}
public class ShellViewModel : PropertyChangedBase, IShell
{
    private readonly IWindowManager windowManager;

    public ShellViewModel()
    {
        this.windowManager = new WindowManager();
        this.windowManager.ShowWindow(new LameViewModel());
    }
}
{{< / highlight >}}

There are two other fields on the ShowWindow method. The third parameter lets you dynamically set the attributes on the Window object.

{{< highlight csharp >}}
public class ShellViewModel : PropertyChangedBase, IShell
{
    private readonly IWindowManager windowManager;

    public ShellViewModel()
    {
        this.windowManager = new WindowManager();

        dynamic settings = new ExpandoObject();
        settings.Height = 600;
        settings.Width = 300;

        this.windowManager.ShowWindow(new LameViewModel(), null, settings);
    }
}
{{< / highlight >}}

*I wish there was more information about working with this on the documentation, but there you have it.*