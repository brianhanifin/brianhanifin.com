---
title: Jekyll Local Testing using Docker on Unraid
date: 2020-09-03 15:00:00 -0800
categories: [Homelab]
tags: [Jekyll, Unraid]
seo:
  date_modified: 2020-09-03 15:00:00 -0800
image: /assets/img/2020-09-03-unraid-docker-jekyll/unraid-docker-jekyll-serve-advanced.png
---

## This site runs a software called [Jekyll](https://jekyllrb.com/)

It requires a copy of the software to be installed on your local computer to preview the article you are writing.
When I started this website I had to install Jekyll on my Windows 10 PC. I didn't like that this required me to
install things like Ruby, RubyGems, GCC, and Make. As infrequently as I write an article, I would forget the
correct command line to run the webserver. Is it `jekyll serve` or `bundler jekyll serve`... ah forget it I just
won't post today.

## Can Jekyll run on Docker?

Docker sure seems like the ideal candidate to run Jekyll. If I could get Jekyll installed in Docker on my Unraid server,
I could point it to my github folder and have it automatically start the webserver.

After some research I discovered [BretFisher's jekyll-serve project on Github](https://github.com/BretFisher/jekyll-serve).
I was able to create a Unraid Docker template which accomplishes my goals!

## Docker Unraid Template

I am sharing my docker template configuration below more for my own reference, but I won't go into any more detail
for now. If you have further questions please ask in the comments at the bottom of this post.

### Basic view
![Partial Unraid docker template screenshot](/assets/img/2020-09-03-unraid-docker-jekyll/unraid-docker-jekyll-serve.png)

### Advanced view
![Partial Unraid docker template screenshot](/assets/img/2020-09-03-unraid-docker-jekyll/unraid-docker-jekyll-serve-advanced.png)

## XML user template

Docker user templates are stored on the Unraid filesystem at `/boot/config/plugins/dockerMan/templates-user/`.

```xml
<?xml version="1.0"?>
<Container version="2">
  <Name>jekyll-serve-brianhanifin.com</Name>
  <Repository>bretfisher/jekyll-serve</Repository>
  <Registry>https://hub.docker.com/r/bretfisher/jekyll-serve/</Registry>
  <Network>bridge</Network>
  <MyIP/>
  <Shell>sh</Shell>
  <Privileged>false</Privileged>
  <Support>https://hub.docker.com/r/bretfisher/jekyll-serve/</Support>
  <Project/>
  <Overview>The Latest Jekyll in a Docker Container For Easy SSG Development.   Converted By Community Applications   Always verify this template (and values) against the dockerhub support page for the container</Overview>
  <Category/>
  <WebUI>http://[IP]:[PORT:4000]/</WebUI>
  <TemplateURL/>
  <Icon>http://brianhanifin.com/assets/img/brian-emoji.png</Icon>
  <ExtraParams/>
  <PostArgs/>
  <CPUset/>
  <DateInstalled>1599166188</DateInstalled>
  <DonateText/>
  <DonateLink/>
  <Description>The Latest Jekyll in a Docker Container For Easy SSG Development.   Converted By Community Applications   Always verify this template (and values) against the dockerhub support page for the container</Description>
  <Networking>
    <Mode>bridge</Mode>
    <Publish>
      <Port>
        <HostPort>4000</HostPort>
        <ContainerPort>4000</ContainerPort>
        <Protocol>tcp</Protocol>
      </Port>
    </Publish>
  </Networking>
  <Data>
    <Volume>
      <HostDir>/mnt/user/media/brianhanifin.com/</HostDir>
      <ContainerDir>/site</ContainerDir>
      <Mode>rw</Mode>
    </Volume>
  </Data>
  <Environment/>
  <Labels/>
  <Config Name="Port" Target="4000" Default="" Mode="tcp" Description="Container Port: 4000" Type="Port" Display="always" Required="false" Mask="false">4000</Config>
  <Config Name="Config" Target="/site" Default="" Mode="rw" Description="Container Path: /site" Type="Path" Display="always" Required="false" Mask="false">/mnt/user/media/brianhanifin.com/</Config>
</Container>
```
