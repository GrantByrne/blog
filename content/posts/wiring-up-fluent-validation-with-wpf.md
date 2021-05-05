---
title: "Wiring Up Fluent Validation With WPF"
date: 2021-05-03T22:30:07-04:00
tags: [csharp, wpf, fluent-validation]
---

**Update**

I originally wrote this proof of concept about 7 years ago early in my time with WPF. I published it on an earlier iteration of my blog via Github Gist and Wordpress ([You can find the gist here](https://gist.github.com/GrantByrne/11243164)). After which point I pretty much forgot out it.

I recently came across the Gist and found that it has helped out a surprising number of people. So, it makes sense to me to pull this in to a blog and annotate it a bit better. If there is interest, I may make some more changes to streamline to examples and take advantage of newer C# features.

## Overview

[Fluent Validation](https://fluentvalidation.net/) is my favorite validation library for C#. It is pretty straightforward to use and it forces you to separate out the validation code into a separate class which IMHO generally makes the code cleaner.

The library includes some pretty standard integrations with ASP.NET, but there never was a first class implementation that integrates a validator with a WPF view model. This post is a proof of concept that I put together to bridge that gap.

## Building the Validator

For the purposes of demonstration. We'll have a UserViewModel which has a property for Name, E-Mail, and Zip Code. So we'll write a quick validator for the usual aspects of that data.

{{< highlight csharp >}}
using System.Text.RegularExpressions;
using FluentValidation;
using WpfFluentValidationExample.ViewModels;

namespace WpfFluentValidationExample.Lib
{
    public class UserValidator : AbstractValidator<UserViewModel>
    {
        public UserValidator()
        {
            RuleFor(user => user.Name)
                .NotEmpty()
                .WithMessage("Please Specify a Name.");

            RuleFor(user => user.Email)
                .EmailAddress()
                .WithMessage("Please Specify a Valid E-Mail Address");

            RuleFor(user => user.Zip)
                .Must(BeAValidZip)
                .WithMessage("Please Enter a Valid Zip Code");
        }

        private static bool BeAValidZip(string zip)
        {
            if (!string.IsNullOrEmpty(zip))
            {
                var regex = new Regex(@"\d{5}");
                return regex.IsMatch(zip);
            }
            return false;
        }
    }
}
{{< / highlight >}}

## Creating the View to Present the Validation

Here is the view that I created to demonstrate this implementation.

This is a standard form written in XAML with a few differences:

- In the property binding for the text on the textboxes we can see that I added a "ValidatesOnDataErrors=True" clause.
- On each one of those textboxes as well, I expanded out the Validation.Error template property with a stack panel which will slot in a validation error when one occurs.

{{< highlight xml >}}
<Window
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:viewModels="clr-namespace:WpfFluentValidationExample.ViewModels" x:Class="WpfFluentValidationExample.Views.UserView"
        Title="UserView" Height="300" MinWidth="500">
    <Window.DataContext>
        <viewModels:UserViewModel/>
    </Window.DataContext>
    <StackPanel>
        <StackPanel Orientation="Horizontal">
            <Label Content="Name" Margin="10"/>
            <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}" Width="200" Margin="10">
                <Validation.ErrorTemplate>
                    <ControlTemplate>
                        <StackPanel Orientation="Horizontal">
                            <AdornedElementPlaceholder x:Name="textBox"/>
                            <TextBlock Margin="10" Text="{Binding [0].ErrorContent}" Foreground="Red"/>
                        </StackPanel>
                    </ControlTemplate>
                </Validation.ErrorTemplate>
            </TextBox>
        </StackPanel>
        <StackPanel Orientation="Horizontal">
            <Label Content="E-Mail" Margin="10"/>
            <TextBox Text="{Binding Email, UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}" Width="200" Margin="10">
                <Validation.ErrorTemplate>
                    <ControlTemplate>
                        <StackPanel Orientation="Horizontal">
                            <AdornedElementPlaceholder x:Name="textBox"/>
                            <TextBlock Margin="10" Text="{Binding [0].ErrorContent}" Foreground="Red"/>
                        </StackPanel>
                    </ControlTemplate>
                </Validation.ErrorTemplate>
            </TextBox>
        </StackPanel>
        <StackPanel Orientation="Horizontal">
            <Label Content="Zip" Margin="10"/>
            <TextBox Text="{Binding Zip, UpdateSourceTrigger=PropertyChanged, ValidatesOnDataErrors=True}" Width="200" Margin="10">
                <Validation.ErrorTemplate>
                    <ControlTemplate>
                        <StackPanel Orientation="Horizontal">
                            <!-- Placeholder for the TextBox itself -->
                            <AdornedElementPlaceholder x:Name="textBox"/>
                            <TextBlock Margin="10" Text="{Binding [0].ErrorContent}" Foreground="Red"/>
                        </StackPanel>
                    </ControlTemplate>
                </Validation.ErrorTemplate>
            </TextBox>
        </StackPanel>
        <Button Margin="10">Submit</Button>
    </StackPanel>
</Window>
{{< / highlight >}}

## The Code Behind on the View

Including this for the sake of completeness. Please ignore.

{{< highlight csharp >}}
using System.Windows;
using WpfFluentValidationExample.ViewModels;

namespace WpfFluentValidationExample.Views
{
    /// <summary>
    /// Interaction logic for UserView.xaml
    /// </summary>
    public partial class UserView : Window
    {
        public UserView()
        {
            InitializeComponent();
            DataContext = new UserViewModel();
        }
    }
}
{{< / highlight >}}

## The Viewmodel

The viewmodel is the integration point between the fluent validator, the data, and the view. Initially this looks like a standard viewmodel. We have a property for the name, email, and zip code.

Closer to the bottom, you'll find the integration for the validation:

- There is the integration of the overloaded [] operator which matches up the property name with the validation.
- There is also the addition of an Error string which combines together all the errors into a single string

{{< highlight csharp >}}
using System;
using System.ComponentModel;
using System.Linq;
using WpfFluentValidationExample.Lib;

namespace WpfFluentValidationExample.ViewModels
{
    public class UserViewModel : INotifyPropertyChanged, IDataErrorInfo
    {
        private readonly UserValidator _userValidator = new UserValidator();

        private string _zip;
        private string _email;
        private string _name;

        public string Name
        {
            get { return _name; }
            set
            {
                _name = value;
                OnPropertyChanged("Name");
            }
        }

        public string Email
        {
            get { return _email; }
            set
            {
                _email = value;
                OnPropertyChanged("Email");
            }
        }

        public string Zip
        {
            get { return _zip; }
            set
            {
                _zip = value;
                OnPropertyChanged("Zip");
            }
        }

        public string this[string columnName]
        {
            get
            {
                var firstOrDefault = _userValidator.Validate(this).Errors.FirstOrDefault(p => p.PropertyName == columnName);
                if (firstOrDefault != null)
                    return _userValidator != null ? firstOrDefault.ErrorMessage : "";
                return "";
            }
        }

        public string Error
        {
            get
            {
                if (_userValidator != null)
                {
                    var results = _userValidator.Validate(this);
                    if (results != null && results.Errors.Any())
                    {
                        var errors = string.Join(Environment.NewLine, results.Errors.Select(x => x.ErrorMessage).ToArray());
                        return errors;
                    }
                }
                return string.Empty;
            }
        }

        public event PropertyChangedEventHandler PropertyChanged;

        protected virtual void OnPropertyChanged(string propertyName)
        {
            PropertyChangedEventHandler handler = PropertyChanged;
            if (handler != null) handler(this, new PropertyChangedEventArgs(propertyName));
        }
    }
}
{{< / highlight >}}

## Conclusion

Thanks for sticking around to the end. I hope this helped you out improving the validation on your WPF application.

If this helped you out, it would be helpful to star the Github Gist that is linked at the top of the post so that I know that I'm helping people out.