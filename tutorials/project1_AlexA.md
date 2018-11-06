id:Project1AlexA

# CS4518 Assignment 1 - By Alexander Antaya
###### By: Alexander Antaya (aantaya)

This is a tutorial that will go over the development of the Petwars application at a high-level. This tutorial is assuming prerequisite knowledge of Android Development and strong knowledge of Java.


## Highlights

This application leverages the relatively new Room Persistence Library, which is part of the Android Jetbrains collection. The Room Persistence Library provides a layer of abstraction to the conventional SQLite database. It removes much of the boilerplate required for setting up an SQLite project and provides a excellent framework for following good Android programming conventions. More information about the Room Persistence Library can be found [here](https://developer.android.com/topic/libraries/architecture/room).

### Prerequisites

* Android Studio installed
* Java environment setup (JDK, etc...)
* AVD or Physical Android device running at least API 19
* Knowledge of Java and Android specific libraries
* Familiarity with the LiveData and ViewModel lifecycle components.

## Setting Up The Backend

Our backend will be very simple. We will have a single database, a data access layer, a single model, and a single ViewModel.

### Database

First, we will need to create an abstract class that will allow us to create our database and access a reference to it. Note that we will be using the singleton pattern here so we ensure that we never accidentally create multiple instances of our database. Also note the annotation in the class declaration. Room uses annotations to add metadata to a Java source file, they can't affect the semantics of a program directly. This allows Room to be able to check SQL queries at compile-time, which can save a lot of development time.

Create this class:

```
@Database(entities = {CatEntity.class}, version = 1)
public abstract class CatDatabase extends RoomDatabase{
    private static volatile CatDatabase INSTANCE;

    public abstract CatDao catDao();

    //Use singleton pattern so we don't accidentally create multiple databases
    static CatDatabase getDatabase(final Context context) {
        if (INSTANCE == null) {
            synchronized (CatDatabase.class) {
                if (INSTANCE == null) {
                    // Create database
                    INSTANCE = Room.databaseBuilder(context.getApplicationContext(),
                            CatDatabase.class, "cat_database")
                            .build();
                }
            }
        }
        return INSTANCE;
    }
}
```

### Data Access Object (DAO)

A Data Access Object (DAO) is an abstract class or interface that contains methods to define database queries. The annotated methods in this class are used to generate the corresponding SQL at compile time. This abstraction helps to reduce the amount of repetitive boilerplate code you need to maintain compared to traditional SQLite. Unlike runtime SQL, these annotated methods are parsed and validated at compile time.

Here we are defining an interface that will outline how we interact with the database.

```
@Dao
public interface CatDao {

    @Insert
    void insert(CatEntity cat);

    @Query("UPDATE cat_table SET votes= votes + 1 WHERE _id =:id")
    void incrementVotes(int id);

    @Query("SELECT * FROM cat_table ORDER BY votes DESC")
    LiveData<List<CatEntity>> getAllCats();
}
```

### Models

Here we will be defining a cat entity. Basically, this is a data-structure that will contain all of the information we will need for an individual cat such as name, image, and description. Note, it is not good practice to store images directly in an SQLite database (although you could as a BLOB) because there is a data through-put limit of the Cursor object that can cause application crashes if the byte array/BLOB is sufficiently large. Rather, we will be storing a relative path to images stored on local storage and access them through that path.


```
@Entity(tableName = "cat_table")
public class CatEntity {

    public static final String TAG = "CAT_ENTITY";

    @PrimaryKey(autoGenerate = true)
    @NonNull
    @ColumnInfo(name = "_id")
    private int _id;

    @NonNull
    @ColumnInfo(name = "imagePath")
    private String imagePath;

    @NonNull
    @ColumnInfo(name = "votes")
    private long votes;

    @NonNull
    @ColumnInfo(name = "name")
    private String name;

    @NonNull
    @ColumnInfo(name = "description")
    private String description;

    //Default Constructor
    public CatEntity(@NonNull String imagePath, @NonNull long votes, @NonNull String name, @NonNull String description) {
        this.imagePath = imagePath;
        this.votes = votes;
        this.name = name;
        this.description = description;
    }

    //You will also need to generate getters and setters for all of the fields
    // for space's sake we will not include this here.
}
```

### LiveData and ViewModels

LiveData is a data holder class that can be observed within a given lifecycle. This means that an Observer can be added in a pair with a LifecycleOwner, and this observer will be notified about modifications of the wrapped data. Basically, we can get real-time updates to our ViewModel as we modify the database. ViewModel is a class that is responsible for preparing and managing the data for an Activity or a Fragment. It also handles the communication of the Activity with the rest of the application.

```
public class CatViewModel extends AndroidViewModel {

    public static final String TAG = "CAT_VIEW_MODEL";

    private CatRepository myCatRepository;
    private LiveData<List<CatEntity>> myAllCats;

    public CatViewModel(@NonNull Application _app) {
        super(_app);
        myCatRepository = new CatRepository(_app);
        myAllCats = myCatRepository.getMyAllCats();
    }

    public LiveData<List<CatEntity>> getMyAllCats() { return this.myAllCats; }
    public void insert(CatEntity _cat) { myCatRepository.insert(_cat); }
}
```

### Repository

A Repository is a class that abstracts access to multiple data sources. A Repository class handles data operations. It provides a clean API to the rest of the app for app data. The reason why we used this in the case is because I was originally planning on using a Firebase server/database along with a local database and this would have made my life easier.

```
public class CatRepository {

    public static final String TAG = "CAT_REPOSITORY";

    private CatDao myCatDao;
    private LiveData<List<CatEntity>> myAllCats;

    public CatRepository(Application _app){
        CatDatabase db = CatDatabase.getDatabase(_app);
        myCatDao = db.catDao();
        myAllCats = myCatDao.getAllCats();
    }

    public LiveData<List<CatEntity>> getMyAllCats(){ return this.myAllCats; };

    public void insert (CatEntity _cat) {
        new insertAsyncTask(myCatDao).execute(_cat);
    }

    private static class insertAsyncTask extends AsyncTask<CatEntity, Void, Void> {

        private CatDao myAsyncTaskDao;

        insertAsyncTask(CatDao dao) { myAsyncTaskDao = dao; }

        @Override
        protected Void doInBackground(final CatEntity... params) {
            myAsyncTaskDao.insert(params[0]);
            return null;
        }
    }

}
```

## Setting Up The Front End

Our front end will also be very simple. Basically we will have two activities, one for viewing cats and another for adding cats. One activity will have a RecyclerView of CardViews of cats.

### MainActivity

Our main activity is the most complicated part of the project.

This is the flow of the MainActivity:
* Upon first run of the application, we will ask for permission to read and write to external storage. We ensure this only is called one time by using the SharedPreferences library.
* In the permissions callback, if the user gives us permission, we will then save the 'pre-loaded' to external storage. We are doing this so that when a TA is grading this, they will have two cats pre-loaded and don't have to add more cats to test.
* We will populate the RecyclerView
* If a user clicks on the Floating Action Button (FAB) then that will trigger an intent that will start the next activity to add a cat to the DB.


```
public class MainActivity extends AppCompatActivity {
    public static final int NEW_CAT_ACTIVITY_REQUEST_CODE = 1;
    public static final int HAVE_PERMISSION = 2;
    public static final String TAG = "MAIN_ACTIVITY";
    public final String PREFS_NAME = "MyPrefsFile";
    public final String FIRST_TIME_STRING = "first_time";

    private CatViewModel catViewModel;

    File file1; // the File to save, append increasing numeric counter to prevent files from getting overwritten.
    File file2;

    SharedPreferences settings;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        settings = getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);

        if (Build.VERSION.SDK_INT >= 23) {
            if (checkSelfPermission(android.Manifest.permission.WRITE_EXTERNAL_STORAGE)
                    == PackageManager.PERMISSION_GRANTED) {
                Log.v(TAG, "Permission is granted");
            } else {
                ActivityCompat.requestPermissions(this,
                        new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE}, 2);
                Log.v(TAG, "Permission is revoked");
            }
        } else { //permission is automatically granted on sdk<23 upon installation
            Log.v(TAG, "Permission is granted");
        }

        catViewModel = ViewModelProviders.of(this).get(CatViewModel.class);

        //This will only execute in the first time the application runs...This will create two cats
        //for TA grading...
        Log.v(TAG, "****Settings value: " + settings.getBoolean(FIRST_TIME_STRING, true));

        if (settings.getBoolean(FIRST_TIME_STRING, true)) {
            //the app is being launched for first time, do something
            Log.d(TAG, "First time");

            String imagePath1;
            String imagePath2;
            AssetManager assetManager = getApplicationContext().getAssets();

            InputStream istr;
            Bitmap bitmap1 = null;
            Bitmap bitmap2 = null;

            //Load in the images from assets
            try {
                istr = assetManager.open("kitten_pictures/kitten_1.jpg");
                bitmap1 = BitmapFactory.decodeStream(istr);
                istr = assetManager.open("kitten_pictures/kitten_2.jpg");
                bitmap2 = BitmapFactory.decodeStream(istr);
            } catch (IOException e) {
                e.printStackTrace();
            }

            File dir = new File(getApplicationContext().getFilesDir()+File.separator+"images");
            dir.mkdir();
            String path = dir.toString();
            long count = dir.listFiles().length+1;
            String fileName1 = "cat" + count + ".jpg";
            count++;
            String fileName2 = "cat" + count + ".jpg";
            Log.v(TAG, "Saving Image = " + fileName1);
            Log.v(TAG, "Saving Image = " + fileName2);
            Log.v(TAG, "Location = " + path);
            OutputStream fOut = null;
            file1 = new File(path, fileName1); // the File to save, append increasing numeric counter to prevent files from getting overwritten.
            file2 = new File(path, fileName2);
            try {
                fOut = new FileOutputStream(file1);
            } catch (FileNotFoundException e) {
                e.printStackTrace();
            }

            if (bitmap1 != null && bitmap2 != null) {
                bitmap1.compress(Bitmap.CompressFormat.JPEG, 85, fOut); // saving the Bitmap to a file compressed as a JPEG with 85% compression rate
                try {
                    fOut = new FileOutputStream(file2);
                } catch (FileNotFoundException e) {
                    e.printStackTrace();
                }
                bitmap2.compress(Bitmap.CompressFormat.JPEG, 85, fOut); // saving the Bitmap to a file compressed as a JPEG with 85% compression rate
            }

            try {
                assert fOut != null;
                fOut.flush(); // Not really required
                fOut.close(); // do not forget to close the stream
            } catch (IOException e) {
                e.printStackTrace();
            }

            imagePath1 = file1.getAbsolutePath();
            imagePath2 = file2.getAbsolutePath();

            Log.v(TAG, "Image1 Path: " + imagePath1);
            Log.v(TAG, "Image2 Path: " + imagePath2);

            CatEntity cat1 = new CatEntity(imagePath1, 0, "Fluffy", "The cutest cat ever!");
            CatEntity cat2 = new CatEntity(imagePath2, 0, "Snuffles", "A really cute cat :) !");

            catViewModel.insert(cat1);
            catViewModel.insert(cat2);

            Toast.makeText(
                    getApplicationContext(),
                    R.string.on_init,
                    Toast.LENGTH_LONG).show();

            // record the fact that the app has been started at least once
            settings.edit().putBoolean(FIRST_TIME_STRING, false).apply();
        }

        final CatListAdapter adapter = new CatListAdapter(this);

        RecyclerView recyclerView = findViewById(R.id.recyclerview);
        recyclerView.setAdapter(adapter);
        recyclerView.setHasFixedSize(true);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        catViewModel.getMyAllCats().observe(this, new Observer<List<CatEntity>>() {
            @Override
            public void onChanged(@Nullable final List<CatEntity> words) {
                adapter.setCats(words);
            }
        });

        FloatingActionButton fab = findViewById(R.id.fab);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent(MainActivity.this, AddCatActivity.class);
                startActivityForResult(intent, NEW_CAT_ACTIVITY_REQUEST_CODE);
            }
        });
    }

    //This is a callback from the dialog presented to the user. The images can only be written to storage
    //  if the user allows us to write to external storage
    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        Log.v(TAG, "Permission Callback, Request Code: " + requestCode);
        for(String s : permissions){ Log.v(TAG, "Permission Callback, permission added : " + s); }
        if(requestCode == 2){
            try {
                MediaStore.Images.Media.insertImage(getContentResolver(), file1.getAbsolutePath(), file1.getName(), file1.getName());
                MediaStore.Images.Media.insertImage(getContentResolver(), file2.getAbsolutePath(), file2.getName(), file2.getName());
            } catch (FileNotFoundException e) {
                e.printStackTrace();
            }
        }
    }

    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        if (requestCode == NEW_CAT_ACTIVITY_REQUEST_CODE && resultCode == RESULT_OK) {
            String imagePath = data.getStringExtra(AddCatActivity.EXTRA_IMAGE_PATH);
            String name = data.getStringExtra(AddCatActivity.EXTRA_NAME);
            String desc = data.getStringExtra(AddCatActivity.EXTRA_DESC);

            CatEntity cat = new CatEntity(imagePath, 0, name, desc);
            catViewModel.insert(cat);
        } else {
            Toast.makeText(
                    getApplicationContext(),
                    R.string.empty_not_saved,
                    Toast.LENGTH_LONG).show();
        }
    }
}
```

### Adapter

An Adapter object acts as a bridge between an AdapterView and the underlying data for that view. The Adapter provides access to the data items. The Adapter is also responsible for making a View for each item in the data set. In our case the Adapter will be used to define the behavior in the RecyclerView and the CardViews that are in it.

```
public class CatListAdapter extends RecyclerView.Adapter<CatListAdapter.CatViewHolder>{

    public static final String TAG = "CAT_LIST_ADAPTER";

    class CatViewHolder extends RecyclerView.ViewHolder {
        private final TextView CatRankView;
        private final TextView CatTitleView;
        private final ImageView CatImageView;
        private final TextView CatDescriptionView;

        private CatViewHolder(View itemView) {
            super(itemView);
            CatRankView = itemView.findViewById(R.id.recyclerview_rank);
            CatTitleView = itemView.findViewById(R.id.recyclerview_title);
            CatImageView = itemView.findViewById(R.id.recyclerview_image);
            CatDescriptionView = itemView.findViewById(R.id.recyclerview_desc);
        }
    }

    private final LayoutInflater mInflater;
    private List<CatEntity> mCats; // Cached copy of Cats

    CatListAdapter(Context context) { mInflater = LayoutInflater.from(context); }

    @Override
    public CatViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View itemView = mInflater.inflate(R.layout.recyclerview_item, parent, false);
        return new CatViewHolder(itemView);
    }

    @Override
    public void onBindViewHolder(CatViewHolder holder, int position) {
        if (mCats != null) {
            CatEntity current = mCats.get(position);
            holder.CatRankView.setText(String.valueOf(position+1));
            holder.CatTitleView.setText(current.getName());
            holder.CatDescriptionView.setText(current.getDescription());

            BitmapFactory.Options options = new BitmapFactory.Options();
            options.inSampleSize = 8;
            final Bitmap b = BitmapFactory.decodeFile(current.getImagePath(), options);

            holder.CatImageView.setImageBitmap(b);
        } else {
            // Covers the case of data not being ready yet.
            holder.CatTitleView.setText("No Cat");
        }
    }

    void setCats(List<CatEntity> Cats){
        mCats = Cats;
        notifyDataSetChanged();
    }

    // getItemCount() is called many times, and when it is first called,
    // mCats has not been updated (means initially, it's null, and we can't return null).
    @Override
    public int getItemCount() {
        if (mCats != null)
            return mCats.size();
        else return 0;
    }

}
```

### AddCatActivity

We also will have another activity which will allow the user to add another cat to the database. The user will have to add an image from their device's gallery (preferably of a cat), name the cat, and add a description of the cat.

Once the user clicks save, the application will redirect to the MainActivity and will attempt to add the cat to the database. If it is unsuccessful (probably because they did not put all of the required information), they will get a Toast telling them an appropriate error message. If they are successful, then the cat image will have been saved to local storage and a path to will be saved in the DB along with the other necessary information.

```
public class AddCatActivity extends AppCompatActivity {

    public static final String EXTRA_IMAGE_PATH = "image_path";
    public static final String EXTRA_NAME = "name";
    public static final String EXTRA_DESC = "desc";
    public static final String TAG = "ADD_CAT_ACTIVITY";

    public static final int PICK_IMAGE = 1;

    private EditText myEditNameView;
    private EditText myEditDescView;
    private ImageView myImageCatView;

    private String imagePath;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.add_cat_activity);

        myEditNameView = findViewById(R.id.edit_cat_name);
        myEditDescView = findViewById(R.id.edit_cat_desc);
        myImageCatView = findViewById(R.id.cat_image);


        final Button upload = findViewById(R.id.upload_image);
        upload.setOnClickListener(new View.OnClickListener() {
            public void onClick(View view) {
                Intent intent = new Intent();
                intent.setType("image/*");
                intent.setAction(Intent.ACTION_GET_CONTENT);
                startActivityForResult(Intent.createChooser(intent, "Select Picture"), PICK_IMAGE);
            }
        });

        final Button button = findViewById(R.id.button_save);
        button.setOnClickListener(new View.OnClickListener() {
            public void onClick(View view) {
                Intent replyIntent = new Intent();
                if (TextUtils.isEmpty(myEditNameView.getText()) || TextUtils.isEmpty(myEditDescView.getText())) {
                    setResult(RESULT_CANCELED, replyIntent);
                } else {
                    String name = myEditNameView.getText().toString();
                    String desc = myEditDescView.getText().toString();

                    replyIntent.putExtra(EXTRA_IMAGE_PATH, imagePath);
                    replyIntent.putExtra(EXTRA_NAME, name);
                    replyIntent.putExtra(EXTRA_DESC, desc);
                    setResult(RESULT_OK, replyIntent);
                }
                finish();
            }
        });
    }

    @Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        if (requestCode == PICK_IMAGE) {
            Uri uri = data.getData();
            try {
                Bitmap bitmap = MediaStore.Images.Media.getBitmap(getContentResolver(), uri);
                myImageCatView.setImageBitmap(bitmap);
                boolean temp = isWriteStoragePermissionGranted();
                Log.v(TAG, "*****Permissions: " + temp);
                if (temp) {
                    // Assume block needs to be inside a Try/Catch block.
                    File dir = new File(getApplicationContext().getFilesDir()+File.separator+"images");
                    dir.mkdir();
                    String path = dir.toString();
                    long count = dir.listFiles().length+1;
                    String fileName = "cat" + count + ".jpg";
                    Log.v(TAG, "Saving Image = " + fileName);
                    Log.v(TAG, "Location = " + path);
                    OutputStream fOut = null;
                    File file = new File(path, fileName); // the File to save, append increasing numeric counter to prevent files from getting overwritten.
                    fOut = new FileOutputStream(file);

                    bitmap.compress(Bitmap.CompressFormat.JPEG, 85, fOut); // saving the Bitmap to a file compressed as a JPEG with 85% compression rate
                    fOut.flush(); // Not really required
                    fOut.close(); // do not forget to close the stream

                    imagePath = file.getAbsolutePath();
                    MediaStore.Images.Media.insertImage(getContentResolver(), file.getAbsolutePath(), file.getName(), file.getName());
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
            Log.v(TAG, "*****USER PICKED AN IMAGE!!!*****");
        }
    }

    public boolean isWriteStoragePermissionGranted() {
        if (Build.VERSION.SDK_INT >= 23) {
            if (checkSelfPermission(android.Manifest.permission.WRITE_EXTERNAL_STORAGE)
                    == PackageManager.PERMISSION_GRANTED) {
                Log.v(TAG, "Permission is granted");
                return true;
            } else {
                ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE}, 2);
                Log.v(TAG, "Permission is revoked");
                return false;
            }
        } else { //permission is automatically granted on sdk<23 upon installation
            Log.v(TAG, "Permission is granted");
            return true;
        }
    }
}
```

## Deployment

In order to deploy this application on a device, connect the device to your computer, enable USB debugging on the device through the developer's options. Next click the 'Run' button in Android Studio and select the connected device as a Deployment Target.

## Acknowledgments

* Android documentation is AWESOME :)
