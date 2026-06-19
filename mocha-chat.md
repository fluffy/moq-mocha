# Abstract 

The specification defined out how to provide messaging in Mocha. 


# Goals

Ths specification defines how Mocha sends messages in Channels between Devices. It allows for messages with text, audio, and video as well as translation to different media types and to different languages. 

Terminology and architecture is defined in mocha-arch. 

# MoQ Chat Naming 

The Moq Tracks that carry chat messages are in a Namespace of:

mocha-v1-<Proivder>-<OrgID>-chat-<TeamID>-<ChannelId>

The tracknsames are:

msg-v1-<DevID>

Groups are NTP time in minutes. 

Inside that Group each Message is in a MoQ Object that are have sequentially increasing ObjectIs starting at 0. 

Messages have an extensible type enum of: None, Text

The  MesgID for a message is formed by hash of full track name followed by GroupID and ObjectID. 

Messgages have a prevousMsg list that has the iMsgID of the mmediatllye preceeding messages(s) in the Channel. 

Message have a timestamp in ms formed from the publisher clock of when the sender thinks the messgae was created. 

# Moq Roster Naming 

mocha-v1-<Proivder>-<OrgID>-roster-<TeamID>-<ChannelId>

The tracknames are:

user-<DevID>

Each object has a JSON providing informaition about mapping of UserID to user in the idendity sytem. 

# Channel Discover 

mocha-v1-<Proivder>-<OrgID>-channel_info-<TeamID>

The tracknames are:

channel_<DevID>

Objects have the names of Channels in the team. 



