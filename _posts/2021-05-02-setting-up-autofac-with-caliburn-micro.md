---
layout: post
title:  "Setting up Autofac with Caliburn Micro v3.2.0 and Autofac v4.8.1"
date:   2018-08-15T22:40:27-04:00
categories: csharp .net caliburn.micro autofac
---
I find myself frequently setting up new projects with Caliburn Micro; however, It isn’t always easy to remember the code to integrate Autofac with the bootstrapper. So, here is the template that I use when creating a new application.

    public class ClientBootstrapper : BootstrapperBase
    {
        private static IContainer Container;

        public ClientBootstrapper()
        {
            this.Initialize();
        }

        protected override void Configure()
        {
            var builder = new ContainerBuilder();

            builder.RegisterType<WindowManager>()
                .AsImplementedInterfaces()
                .SingleInstance();

            builder.RegisterType<EventAggregator>()
                .AsImplementedInterfaces()
                .SingleInstance();
                
            Container = builder.Build();
        }

        protected override IEnumerable<object> GetAllInstances(Type service)
        {
            var type = typeof(IEnumerable<>).MakeGenericType(service);
            return Container.Resolve(type) as IEnumerable<object>;
        }

        protected override object GetInstance(Type service, string key)
        {
            if (string.IsNullOrWhiteSpace(key))
            {
                if (Container.IsRegistered(service))
                    return Container.Resolve(service);
            }
            else
            {
                if(Container.IsRegisteredWithKey(key, service))
                    return Container.ResolveKeyed(key, service);
            }

            var msgFormat = "Could not locate any instances of contract {0}.";
            var msg = string.Format(msgFormat, key ?? service.Name);
            throw new Exception(msg);
        }

        protected override void BuildUp(object instance)
        {
            Container.InjectProperties(instance);
        }
    }

Hope this helps you out! Good luck with your WPF application.