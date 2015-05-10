Waterfall
=========
#### Goal

Be able to chain ruby commands, and treat them like a flow.

#### Example

Lets check with the following example, with many different outcomes, which code is not that obvious to follow:

    def submit_application
      if current_user.confirmed?
        if params[:terms_of_service]
          application = current_user.build_application
          if application.save
            render json: { ok: true }
          else 
            render_errors application.errors.full_messages
          end
        else
          render_errors 'You must accept the terms'
        end
      else
        render_errors 'You need to confirm your account first'
      end
    end
    
    def render_error(errors)
      render json: { errors: Array(errors) }, status: 422
    end

Waterfall lets you write it this way:

    def accept_group_terms
      Wf.new
        .when_falsy { current_user.confirmed? }
          .dam { 'You need to confirm your account first' }
        .when_falsy { params[:terms_of_service] }
          .dam { 'You must accept the terms' }
        .chain { @application = current_user.build_application }
        .when_falsy { @application.save }
          .dam { @application.errors.full_messages }
        .chain {  render json: { ok: true } }
        .on_dam { |errors| render json: { errors: Array(errors) }, status: 422 }
    end

Once the flow faces a `dam`, all following instructions are skipped, only the `on_dam` blocks are executed.

This is maybe just fancy, but the real power of waterfall is: you can chain waterfalls, and this is where it begins to be great.

See examples:
- https://gist.github.com/apneadiving/b1e9d30f08411e24eae6
- https://gist.github.com/apneadiving/f1de3517a727e7596564

#### Chaining Waterfalls

    class AcceptInvitation
      include Waterfall
      include ActiveModel::Validations
      
      attr_reader :token, :referrer
      
      validates :referrer, presence: true
      
      def initialize(referrer_id, token)
        @referrer_id, @token = referrer_id, token
      end
      
      def call
        self
          .chain { @referrer = User.find_by(id: @referrer_id) }
          .when_falsy { valid? }
            .dam { errors }
          .chain_wf(invitee: :authenticated_user) do
            AuthenticateFromInvitationToken.new(token)
          end
          .chain do |outflow|
            referrer.complete_affiliation_for(outflow[:invitee])
          end
      end
    end

    class AuthenticateFromInvitationToken
      include Waterfall
      include ActiveModel::Validations
      attr_reader :token, :user
      
      validates :user, presence: true
      
      def initialize(token)
        @token = token
      end
      
      def call
        self 
          .chain { @user = User.from_invitation_token(token) }
          .when_falsy { valid? }
            .dam { errors }
          .chain(:authenticated_user) { user }
      end
    end
    
    # in controller
    def accept_invitation
      Wf.new
        .chain_wf(invitee: :invitee) { AcceptInvitation.new(params[:referrer_id], params[:token]) }
        .chain do |outflow|
          @invitee = outflow[:invitee]
          render :invitation_form
        end
        .on_dam do |errors|
          errors.full_messages.each {|error| flash[:error] = error }
          redirect_to root_path
        end
    end



#### Rationale
Coding is all about writing a flow of commands.

Generally you basically go on, unless something wrong happens. Whenever this happens you have to halt the flow and send feedback to the user.

When conditions stack up, readability decreases. One way to solve it is to create abstractions (service objects or the like). Some gems suggest a nice approach like [light service](https://github.com/adomokos/light-service) and [interactor](https://github.com/collectiveidea/interactor).

I like these approaches, but I dont like to have to write a class each time I need to chain services. Or I even dont want to create a class each time I need to chain something.

My take on this was to create `waterfall`.

Thanks
=========
Huge thanks to [laxrph10](https://github.com/laxrph10) for the help during infinite naming brainstorming.
