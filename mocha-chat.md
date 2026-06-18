# Abstract 

The specification defined out how to provide messaging in Mocha. 


# Goals

Ths specification defines how Mocha sends messages in Channels between Devices. It allows for messages with text, audio, and video as well as translation to different media types and to different languages. 

Terminology and architecture is defined in mocha-arch. 

# MoQ Naming 

The Moq Track are in a Namespace of:

mocha-v1-<Proivder>-<OrgID>-<TeamID>-<ChannelID>-chat

The tracknsames are:

msg-v1-<DevID>

Groups are NTP time in minutes. 

Inside that Group each Message is in a MoQ Object that are have sequentially increasing ObjectIs starting at 0. 

Messages have a type




