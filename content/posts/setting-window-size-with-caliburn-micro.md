---
title: "Setting Window Size With Caliburn Micro"
date: 2021-05-04T22:05:27-04:00
tags: [csharp, wpf, caliburn-micro]
---

This is something that has actually bugged me for a while. Once I figured it out, it annoyed me that I didn't figure it out sooner.

When displaying a window in caliburn, you can set attributes about the Window object when calling it.

**So, lets say you want to set the height and width on the window to 600 x 300:**

First, you would start with something like this:

    public class ShellViewModel : PropertyChangedBase, IShell
    {
        private readonly IWindowManager windowManager;

        public ShellViewModel()
        {
            this.windowManager = new WindowManager();
            this.windowManager.ShowWindow(new LameViewModel());
        }
    }

There are two other fields on the ShowWindow method. The third parameter lets you dynamically set the attributes on the Window object.

    public class ShellViewModel : PropertyChangedBase, IShell
    {
        private readonly IWindowManager windowManager;

        public ShellViewModel()
        {
            this.windowManager = new WindowManager();

            dynamic settings = new ExpandoObject();
            settings.Height = 600;
            settings.Width = 300;
            settings.SizeToContent = SizeToContent.Manual;

            this.windowManager.ShowWindow(new LameViewModel(), null, settings);
        }
    }

*I wish there was more information about working with this on the documentation, but there you have it.*