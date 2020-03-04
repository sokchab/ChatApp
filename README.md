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
```
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
```

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
```

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
```
class Chat extends Model
{
    protected $table= 'chats';


    protected $fillable = [
        'user',
        'message',
    ];
}

```
```
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
