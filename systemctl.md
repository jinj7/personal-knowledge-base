
# 1. Introductions

In systemd, the target of most actions are “units”, which are resources that systemd knows how to manage. Units are categorized by the type of resource they represent and they are defined with files known as unit files. The type of each unit can be inferred from the suffix on the end of the file.

For service management tasks, the target unit will be service units, which have unit files with a suffix of .service. However, for most service management commands, you can actually leave off the .service suffix, as systemd is smart enough to know that you probably want to operate on a service when using service management commands.

&nbsp;


# 2. Service management

```sh
systemctl start application.service

systemctl stop application.service

systemctl status application.service

systemctl reload application.service

systemctl restart application.service

systemctl disable application.service

systemctl is-active application.service
```

&nbsp;


# 3. Service management

## 3.1 Listing current units

```sh
systemctl list-units

systemctl list-units --all

systemctl list-units --all --state=inactive

systemctl list-units --type=service
```

## 3.2 Listing all unit files

```sh
systemctl list-unit-files
```

&nbsp;


# 4. Unit management

## 4.1 Displaying a unit file

```sh
systemctl cat application.service
```

## 4.2 Displaying dependencies

```sh
systemctl list-dependencies application.service

systemctl list-dependencies application.service --reverse
```

&nbsp;


# 5. Reference

<https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units>

<http://www.jinbuguo.com/systemd/systemd.html>

&nbsp;
