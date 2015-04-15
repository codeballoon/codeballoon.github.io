---
layout:     post
title:      "Understanding Phone Number Masking"
date:       2015-04-11 11:30:00
categories: programming ruby
---
If you've used Uber, Instacart, or even Google Voice, then you know what phone number masking is and probably have a rough idea how you'd implement it. Just for fun, I'm going to take some time and explore a simple Ruby on Rails implementation.

Problem
=======

So, I'm launching a new app, Fisherman's Helper. When you're out in the middle of the lake and run out of bait, just pull open the Fisherman's Helper native app and push the big red button -- a Helper will row out to your location with a new can of bait. I'll probably support texting a 1-800 number too, for fishermen without an iOS or Android device, but they'll have to punch the GPS coordinates displayed on their fish finder into the text message.

Now, whether the fisherman requests a helper via native app or text, a few minutes later I'd like the fisherman to receive a text message from a masked phone number saying "I'm on my way." The fisherman could then text back and forth with this masked number (the helper) for some duration -- at least the time it takes the helper to get there, and maybe an additional 15-30 minutes afterwards as well. On the other end, any texts the helper receives should also be masked.

Implementation
==============

Let's assume I went out and purchased 10 numbers from [Twilio][twilio], and stashed them in a new table called `masked_numbers`. I also put together a table called `masked_conversations`, containing 3 columns: `from_number`, `masked_to_number`, `actual_to_number`. With this table in hand, I can start a skeleton controller for receiving text messages from Twilio.

{% highlight ruby %}
class TextMessagesController < ApplicationController
  def incoming
    conversation = MaskedConversation.where(:from_number => params['From'], :masked_to_number => params['To']).first

    if conversation
      # Send a new text message to conversation.actual_to_number containing the message from params['Body']
    else
      render :text => "We're sorry, this conversation has ended. Thank you for using Fisherman's Helper!", :content_type => 'text/plain'
    end
  end
end
{% endhighlight %}

So, before we can forward the message to the intended destination, we need to figure out what the return number (that is, the _masked_
return number) is. It might already exist, but we may need to create a new one. I'm going to encapsulate that in the MaskedConversation model.

{% highlight ruby %}
class MaskedConversation < ActiveRecord::Base
  attr_accessible :from_number, :masked_to_number, :actual_to_number

  class << self
    def find_or_create_conversation(from_number, actual_to_number)
      MaskedConversation.where(:from_number => from_number, :actual_to_number => actual_to_number).first ||
      MaskedConversation.create!(:from_number => from_number, :actual_to_number => actual_to_number)
    end

    def send_message(from_number, to_number, message)
      masked_from_number = MaskedConversation.find_or_create_conversation(to_number, from_number).masked_to_number
      Twilio.send_message(masked_from_number, to_number, message)
    end
  end
end
{% endhighlight %}

Now `find_or_create_conversation` does create a new conversation, but it doesn't actually generate an appropriate masked number. We can add
that as a callback on the model.

{% highlight ruby %}
class MaskedConversation < ActiveRecord::Base

  before_save :generate_masked_to_number

  def generate_masked_to_number
    return if masked_to_number

    existing_conversations = MaskedConversation.where(:from_number => from_number).pluck(:masked_to_number)

    if existing_conversations.any?
      self.masked_to_number = MaskedNumber.where("phone_number NOT IN (?)", existing_conversations).order("RAND()").first
    else
      self.masked_to_number = MaskedNumber.order("RAND()").first
    end

    raise "No masked numbers available" unless self.masked_to_number
  end
end
{% endhighlight %}


Scaling
=======

The beautiful thing about number masking is that while you do need a pool of numbers to draw from, your pool doesn't necessarily have to scale
directly with your number of users. Instead, it scales with the number of unique conversations _per user_ that your application needs to support at one time.

In my case, I'd expect fishermen to use my app somewhere between 0 and 3 times per day. So even if my wildly successful app is used by a million different users every day, my pool of 10 numbers is still plenty big enough.

On the other side, a busy helper can probably row bait out to 3-4 fishermen per hour (even more if you do multiple boats in one trip). If we assume that we'll expire a conversation one hour after the bait is delivered, then a given helper may have 10-14 active conversations at one time. He's not likely to be talking to all those people, of course, but the application would need to be ready in case he texted any of those numbers. So, to be safe, I might want to bump my number pool up to 20 numbers. Again, though, if I grow from 10 helpers up to 100 helpers, my pool is still just fine.

In the future I might decide to keep conversations open for 2 or 3 hours after delivery, which _would_ force my pool to grow. For future-proofing, it would make sense to build some methods into my `MaskedNumber` model that can grow or shrink the number pool by making the appropriate Twilio API calls.


Other Variations
================

If you've used Google Voice for sending text messages, or as a support line, then you've seen the one-sided variation -- your users can all text the same 1-800 number for support, but individual customer service reps can receive those texts and carry on conversations with customers. You could imitate this behavior for users by making some changes to the controller, as shown below.

{% highlight ruby %}
class TextMessagesController < ApplicationController
  def incoming
    conversation = MaskedConversation.where(:from_number => params['From'], :masked_to_number => params['To']).first

    if conversation
      MaskedConversation.send_message(params['From'], conversation.actual_to_number, params['Body'])
    elsif params['To'] == COMPANY_SUPPORT_LINE
      # A brand-new text message from a user to the company line goes to a random CSR
      csr_phone_number = CustomerServiceRep.order("RAND()").first.phone_number
      MaskedConversation.send_message(params['From'], csr_phone_number, params['Body'])
      MaskedConversation.create(:from_number => params['From'], :masked_to_number => COMPANY_SUPPORT_LINE, :actual_to_number => csr_phone_number)
    else
      render :text => "Conversation has ended.", :content_type => "text/plain"
    end
  end
end
{% endhighlight %}

Note that when a brand-new text comes in, not only do we invent a new masked number and forward the message to a random representative, we also store an entry in the `masked_conversations` table linking that user's phone number to the representative. That way if the customer opens up a support conversation with two text messages back-to-back, they won't end up going to two different CSRs. This also ensures that if the CSR replies, when we forward it to the customer, we won't invent a new masked number (we want all the texts the user receives to look like they are coming from the primary support line, regardless of which representative is responding to it.)


Expiring Conversations
======================

At some point you'll need to expire old conversations. Depending on your use case this might be minutes, hours, or days after a given "transaction" ends in your application. Adding an `expire_at` column to the conversations table, along with a simple script to delete rows that are ready to expire, would be one way to do this.

You also could base this not on the transaction, but on actual uses of the masked number.  For example, if you bumped `expire_at` further back every time you retrieved a `MaskedConversation`, then conversations that are still active could stay alive indefinitely.

One last option would be to avoid MySQL altogether and store information about ongoing conversations in [Redis][redis], making use of `EXPIRE` instead of an expiration script. I'll leave the code for that one up to you!

[twilio]:      http://www.twilio.com
[redis]:       http://redis.io/
