# ChatApp

### ENV
```
BROADCAST_DRIVER=pusher

PUSHER_APP_ID=your id
PUSHER_APP_KEY=your key
PUSHER_APP_SECRET=your secret
PUSHER_APP_CLUSTER=your cluster
```

### Migrations
```php
class CreateUsersTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('users', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->string('photo')->default('avatar1.png');
            $table->rememberToken();
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('users');
    }
}
```

```php
class CreateChatsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('chats', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('user');
            $table->text('message');
            $table->timestamps();

            $table->foreign('user')->references('id')->on('users')->onDelete('cascade')->onUpdate('cascade');
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('chats');
    }
}
```

```php
class CreateVChatTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        $sql = "CREATE VIEW v_chat as 
                SELECT c.user, u.name, c.message, c.created_at, c.updated_at, u.photo
                FROM chats c INNER JOIN users u ON c.user = u.id;";

        DB::statement($sql);
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        DB::statement("drop view if exists v_chat;");
    }
}
```

## Model & Controller
```php
class Chat extends Model
{
    protected $table= 'chats';
    protected $fillable = [
        'user',
        'message',
    ];
}
```

```php
use Illuminate\Support\Facades\DB;
use App\Chat;
use App\Events\ChatEvent;

class ChatController extends Controller
{
    public function __construct(){
        $this->middleware('auth');
    }

    public function message(){
        $results = DB::table('v_chat')->orderBy('created_at', 'asc')->get();

        $messages = array();

        foreach($results as $row){
            $chat = array();

            if($row->user === auth()->user()->id){
                $chat['user'] = "You";
                $chat['flag'] = false;
            }else{
                $chat['user'] = $row->name;
                $chat['flag'] = true;
            }
            $chat['message'] = $row->message;
            $chat['created_at'] = \Carbon\Carbon::parse($row->created_at)->diffForHumans();
            $chat['photo']  = $row->photo;

            $messages[] = $chat;
        }

        return response()->json($messages);
    }

    public function save(Request $r){
        $r->validate([
            'message' => 'required'
        ]);

        $chat = Chat::create([
            'user' => auth()->user()->id,
            'message' => $r->message,
        ]);
        

        event(new ChatEvent);

        return response()->json($chat);
    }
}
```

### Events
```php

class ChatEvent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new PrivateChannel('channel-chat');
    }
}
```

## Route
```php
// web.php
Route::get('/', function () {
    return view('welcome');
});
Auth::routes();
Route::get('/home', 'HomeController@index')->name('home');
Route::get('/message', 'ChatController@message')->name('message');
Route::post('/message', 'ChatController@save')->name('message.save');
```

```php
//channels.php
Broadcast::channel('App.User.{id}', function ($user, $id) {
    return (int) $user->id === (int) $id;
});
Broadcast::channel('channel-chat', function(){
    return true;
});
```

## js
```javascript
// app.js
require('./bootstrap');

window.Vue = require('vue');

import Vue from 'vue'
import { Form, HasError, AlertError } from 'vform';
 
Vue.component(HasError.name, HasError);
Vue.component(AlertError.name, AlertError);

window.Form = Form;
window.Fire = new Vue();
 
import VueChatScroll from 'vue-chat-scroll';
Vue.use(VueChatScroll);

Vue.component('home', require('./components/Home.vue').default);
Vue.component('message', require('./components/Message.vue').default);

const app = new Vue({
    el: '#app',
});


// bootstrap.js
window._ = require('lodash');

try {
    window.Popper = require('popper.js').default;
    window.$ = window.jQuery = require('jquery');

    require('bootstrap');
    require('admin-lte');
} catch (e) {}

window.axios = require('axios');

import Echo from 'laravel-echo';

window.Pusher = require('pusher-js');

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: process.env.MIX_PUSHER_APP_KEY,
    cluster: process.env.MIX_PUSHER_APP_CLUSTER,
    encrypted: true
});
```

## Components
```vue
//Home.vue
<template>
    <div class="container">
        <div class="row justify-content-center">
            <div class="col-md-8">
                <div class="card direct-chat direct-chat-primary">
                    <div class="card-header">Dashboard</div>
                    <div class="card-body" style="max-height: 400px" v-chat-scroll>
                        
                        <message v-for="chat,index in chats" :chats="chat" :key="index">
                        </message>

                    </div>
                    <div class="card-footer">
                        <form @submit.prevent="send" method="post">
                        <div class="input-group">
                            <input type="text" name="message" v-model="form.message" placeholder="Type Message ..." class="form-control" required>
                            <span class="input-group-append">
                            <button type="submit" class="btn btn-primary">Send</button>
                            </span>
                        </div>
                        </form>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    export default {
        data(){
            return{
                chats: [],
                form: new Form({
                    message: ""
                })
            }
        },
        mounted() {
            this.get_chat();

            Fire.$on('onSend', ()=>{
                this.get_chat();
            });

            Echo.private('channel-chat')
                .listen('ChatEvent', (e)=>{
                    Fire.$emit('onSend');
                });
        },
        methods: {
            get_chat(){
                axios.get('message')
                    .then(res => {
                        this.chats = res.data;
                    }).catch(err => console.log(err));
            },
            send(){
                this.form.post('message')
                    .then(res => {
                        this.form.reset();
                        Fire.$emit('onSend');
                    }).catch(err => console.log(err));
            }
        }
    }
</script>



//Message.vue
<template>
    <div class="direct-chat-messages" style="height: auto">
        <div class="direct-chat-msg" :class="chats.flag ? 'right': ''">
            <div class="direct-chat-infos clearfix">
                <span class="direct-chat-name" :class="chats.flag ? 'float-right' : 'float-left'">{{ chats.user }}</span>
                <span class="direct-chat-timestamp" :class="chats.flag ? 'float-left': 'float-right'">{{ chats.created_at }}</span>
            </div>
            <img class="direct-chat-img" :src="'/img/' + chats.photo" alt="message user image">
            <div class="direct-chat-text">
                {{ chats.message }}
            </div>
        </div>
    </div>
</template>
<script>
    export default {
        props: ['chats']
    }
</script>
```

## Config
```php
 //app.php
 'providers' => [
        /*
         * Application Service Providers...
         */
        App\Providers\AppServiceProvider::class,
        App\Providers\AuthServiceProvider::class,
        App\Providers\BroadcastServiceProvider::class,
        App\Providers\EventServiceProvider::class,
        App\Providers\RouteServiceProvider::class,

  ],
```
