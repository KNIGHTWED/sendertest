# sendertest

MainActivity.java

```java
package com.samsung.multiscreen.msf20.helloworld;

import android.app.AlertDialog;
import android.content.DialogInterface;
import android.os.Bundle;
import android.support.v4.view.MenuItemCompat;
import android.support.v7.app.ActionBarActivity;
import android.text.TextUtils;
import android.util.Log;
import android.view.KeyEvent;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.view.View.OnClickListener;
import android.view.inputmethod.EditorInfo;
import android.widget.EditText;
import android.widget.ListView;
import android.widget.TextView;
import android.widget.TextView.OnEditorActionListener;
import android.widget.Toast;

import com.samsung.multiscreen.Application;
import com.samsung.multiscreen.Client;
import com.samsung.multiscreen.Error;
import com.samsung.multiscreen.Result;
import com.samsung.multiscreen.Service;
import com.samsung.multiscreen.msf20.sdk.ServiceWrapper;
import com.samsung.multiscreen.util.RunUtil;

/**
 * @author plin
 *
 * Main Hello World activity.
 * 
 */
public class MainActivity extends ActionBarActivity {

    private static final String TAG = new Object() {}.getClass().getEnclosingClass().getName();

    private Menu connectMenu;
    private View connectIconActionView;
    
    private AlertDialog listDialog;

    private static ActionBarConnectIcon connectIcon;

    private App app;
    private HelloWorldWebApplicationHelper msHelloWorld;

    private SearchListener searchListener = new SearchListener() {
        
        @Override
        public void onStop() {
            Log.d(TAG, "Search.onStop()");
            invalidateMenu();
            if (connectMenu != null) {
                MenuItem item = connectMenu.findItem(R.id.action_refresh);
                if (MenuItemCompat.getActionView(item) != null) {
                    MenuItemCompat.setActionView(item, null);
                }
                if (!item.isEnabled()) {
                    item.setEnabled(true);
                }
            }
        }
        
        @Override
        public void onStart() {
            Log.d(TAG, "Search.onStart()");
        }

        @Override
        public void onLost(Service service) {
//            Log.d(TAG, "Search.onLost(): " + service.toString());
            invalidateMenu();
        }

        @Override
        public void onFound(Service service) {
//            Log.d(TAG, "Search.onFound(): " + service.toString());
            invalidateMenu();
        }
    };
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        app = App.getInstance();
        msHelloWorld = app.getHelloWorldWebApplication();

        validateServices(new Result<Boolean>() {

            @Override
            public void onError(Error error) {
            }

            @Override
            public void onSuccess(Boolean result) {
                msHelloWorld.startDiscovery(searchListener);
            }
        });
        
        final EditText editText = (EditText) findViewById(R.id.sendText);
        editText.setOnEditorActionListener(new OnEditorActionListener() {
            @Override
            public boolean onEditorAction(TextView v, int actionId, KeyEvent event) {
                boolean handled = false;
                if (actionId == EditorInfo.IME_ACTION_SEND) {
                    String text = v.getText().toString();
                    if (!TextUtils.isEmpty(text)) {
                            
                        Application application = msHelloWorld.getApplication();
                        
                        if ((application != null) && application.isConnected()) {
                            application.publish("say", text);
                        }
                    }
                    handled = true;
                }
                return handled;
            }
        });
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        Log.d(TAG, "onCreateOptionsMenu");
        connectMenu = menu;
        // Inflate the menu; this adds items to the action bar if it is present.
        getMenuInflater().inflate(R.menu.main, menu);

        MenuItem item = connectMenu.findItem(R.id.action_connect);
        connectIconActionView = MenuItemCompat.getActionView(item);
        connectIcon = new ActionBarConnectIcon(connectIconActionView, ActionBarConnectIcon.LIGHT_THEME, connectIconOnClickListener);

        item = connectMenu.findItem(R.id.action_refresh);
        if (!msHelloWorld.isRunning()) {
            item.setEnabled(true);
        } else {
            item.setEnabled(false);
            MenuItemCompat.setActionView(item, R.layout.refresh);
        }

        return super.onCreateOptionsMenu(menu);
    }

    @Override
    public boolean onPrepareOptionsMenu(Menu menu) {
        Log.d(TAG, "onPrepareOptionsMenu");
        invalidateMenu();
        return super.onPrepareOptionsMenu(menu);
    }

    @Override
    public boolean onOptionsItemSelected(final MenuItem item) {
        // Handle action bar item clicks here. The action bar will
        // automatically handle clicks on the Home/Up button, so long
        // as you specify a parent activity in AndroidManifest.xml.
        int id = item.getItemId();
        if (id == R.id.action_connect) {
            showServices();
        } else if (id == R.id.action_refresh) {
            item.setEnabled(false);
            MenuItemCompat.setActionView(item, R.layout.refresh);

            validateServices(new Result<Boolean>() {

                @Override
                public void onError(Error error) {
                }

                @Override
                public void onSuccess(Boolean result) {
                    msHelloWorld.startDiscovery(searchListener);
                }
            });
        }
        return super.onOptionsItemSelected(item);
    }

    private void invalidateMenu() {
        updateConnectIcon();
    }
    
    @Override
    public void onStart() {
        Log.d(TAG, "onStart: " + TAG);
        super.onStart();
    }

    @Override
    public void onResume() {
        Log.d(TAG, "onResume: " + TAG);
        super.onResume();
        invalidateMenu();
    }

    @Override
    public void onPause() {
        Log.d(TAG, "onPause: " + TAG);
        super.onPause();
        invalidateMenu();
    }

    @Override
    protected void onStop() {
        Log.d(TAG, "onStop: " + TAG);
        super.onStop();
    }

    @Override
    public void onDestroy() {
        Log.d(TAG, "onDestroy: " + msHelloWorld);
        msHelloWorld.cleanup();
        super.onDestroy();
    }

    private OnClickListener connectIconOnClickListener = new OnClickListener() {
        
        @Override
        public void onClick(View v) {
            
            showServices();
        }
    };
    
    private void updateConnectIcon() {
        
        EditText editText = (EditText) findViewById(R.id.sendText);
        if (connectIcon != null) {
            Application application = msHelloWorld.getApplication();
            if ((application != null) && application.isConnected()) {
                connectIcon.setConnected();
                editText.setEnabled(true);
                return;
            }

            if (msHelloWorld.getServiceListAdapter().getCount() > 0) {
                connectIcon.setEnabled();
                editText.setEnabled(false);
                return;
            }

            // If we fall through, then no services were found.
            connectIcon.setDisabled();
        }
        editText.setEnabled(false);
    }
    
    /**
     * Show the list of services found during discovery. 
     */
    private void showServices() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this, AlertDialog.THEME_HOLO_LIGHT);

        ServiceWrapper selectedWrapper = msHelloWorld.getService();
        final ServiceListAdapter serviceListAdapter = msHelloWorld.getServiceListAdapter();
        
        builder
            .setTitle(R.string.services_title)
            .setSingleChoiceItems(serviceListAdapter, 
                    serviceListAdapter.getPosition(selectedWrapper), 
                    new DialogInterface.OnClickListener() {
    
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    listDialog.dismiss();
                    
                    int selected = serviceListAdapter.getPosition(msHelloWorld.getService());
                    final ServiceWrapper wrapper = serviceListAdapter.getItem(which);
                    final Service service = wrapper.getService();
                    final Application msApplication = msHelloWorld.getApplication();

                    // If already connected to the selected service, then 
                    // disconnect. Otherwise, ensure that we disconnect the 
                    // previous connection.
                    Result<Client> result = null;
                    if ((which == selected) && 
                            (msApplication != null) && 
                            msApplication.isConnected()) {
                        result = new Result<Client>() {

                            @Override
                            public void onSuccess(Client client) {
                                RunUtil.runOnUI(new Runnable() {

                                    @Override
                                    public void run() {
                                        invalidateMenu();
                                    }
                                });
                            }

                            @Override
                            public void onError(Error error) {
                                RunUtil.runOnUI(new Runnable() {

                                    @Override
                                    public void run() {
                                        invalidateMenu();
                                    }
                                });
                            }
                        };
                    } else if (which != selected) {

                        final Runnable runnable = new Runnable() {
                            
                            @Override
                            public void run() {
                                // Show icon connecting animation 
                                showConnecting();
                                
                                msHelloWorld.connectAndLaunch(wrapper, 
                                    launchCallback, 
                                    new ChannelListener() {
                                        
                                        private void resetService() {
                                            msHelloWorld.setService(null);
                                            invalidateMenu();
                                            if (listDialog != null) {
                                                ListView listView = listDialog.getListView();
                                                listView.setItemChecked(listView.getCheckedItemPosition(), false);
                                            }
                                        }
                                        
                                        @Override
                                        public void onConnect(Client client) {
                                        }

                                        @Override
                                        public void onDisconnect(Client client) {
                                            invalidateMenu();
                                            resetService();
                                        }

                                        @Override
                                        public void onClientConnect(Client client) {
                                            Log.d(TAG, "ClientConnect: " + client.toString());
                                            if (client.isHost()) {
                                                invalidateMenu();
                                            }
                                        }

                                        @Override
                                        public void onClientDisconnect(Client client) {
                                            Log.d(TAG, "ClientDisconnect: " + client.toString());
                                            if (client.isHost()) {
                                                resetService();
                                            }
                                        }
                                        
                                        @Override
                                        public void onReady() {
                                            Log.d(TAG, "onReady");
                                            showMessage("Ready, set, go!");
                                        }
                                    }
                                );
                            }
                        };
                        
                        if ((msApplication != null) && 
                                msApplication.isConnected()) {
                            result = new Result<Client>() {

                                private void launch() {
                                    RunUtil.runInBackground(runnable);
                                }
                                
                                @Override
                                public void onSuccess(Client client) {
                                    Log.d(TAG, "onSuccess: " + client.toString());
                                    launch();
                                }

                                @Override
                                public void onError(Error error) {
                                    Log.d(TAG, "onError: " + error.toString());
                                    launch();
                                }
                            };
                        } else {
                            RunUtil.runInBackground(runnable);
                        }
                    }
                    
                    if (result != null) {
                        msHelloWorld.resetChannel(result);
                    }
                }
            })
            .setOnCancelListener(new DialogInterface.OnCancelListener() {
    
                @Override
                public void onCancel(DialogInterface dialog) {
                    Log.d(TAG, "Canceled");
                }
            });
        listDialog = builder.create();
        listDialog.show();
    }

    /**
     * Show the connecting icon while the device is connecting to the selected 
     * service.
     */
    private void showConnecting() {
        RunUtil.runOnUI(new Runnable() {
            
            @Override
            public void run() {
                if (connectIcon != null) {
                    connectIcon.setConnecting();
                }
            }
        });
    }
    
    private void showMessage(String message) {
        Toast.makeText(this, 
                message, 
                Toast.LENGTH_LONG).show();
    }

    private Result<Client> launchCallback = new Result<Client>() {

        @Override
        public void onSuccess(Client client) {
            Log.d(TAG, "launch Success: " + client.toString());
            
            RunUtil.runOnUI(new Runnable() {
                public void run() {
                    invalidateMenu();
                }
            });
        }

        @Override
        public void onError(Error error) {
            RunUtil.runOnUI(new Runnable() {
                public void run() {
                    msHelloWorld.setService(null);
                    invalidateMenu();
                    showMessage(app.getConfig().getString(R.string.launch_err));
                }
            });
        }
    };
    
    private class ValidateHelper {
        final int numServices;
        private int numReturned = 0;
        
        ValidateHelper(int numServices) {
            this.numServices = numServices;
        }
        
        boolean hasAllReturned() {
            return (numReturned >= numServices);
        }
        
        void hasReturned() {
            numReturned++;
        }
    }
    
    private void validateServices(final Result<Boolean> callback) {
        
        final ServiceListAdapter adapter = msHelloWorld.getServiceListAdapter();
        final int numServices = adapter.getCount();
        
        if (numServices > 0) {
            final ValidateHelper helper = new ValidateHelper(numServices);
            for (int i = 0; i < numServices; i++) {
                final ServiceWrapper wrapper = adapter.getItem(i);
                Service service = wrapper.getService();
                Service.getByURI(service.getUri(), 2000, new Result<Service>() {

                    @Override
                    public void onSuccess(Service service) {
                        Log.d(TAG, "validateServices onSuccess(): " + service.toString());
                        // We can contact the service, so keep it in the master 
                        // list.
                        helper.hasReturned();
                        if (helper.hasAllReturned()) {
                            callback.onSuccess(true);
                        }
                    }

                    @Override
                    public void onError(Error error) {
                        Log.d(TAG, "validateServices onError() unable to contact service: " + error.toString());
                        helper.hasReturned();
                        adapter.remove(wrapper);
                        invalidateMenu();
                        if (helper.hasAllReturned()) {
                            callback.onSuccess(true);
                        }
                    }
                });
            }
        } else {
            callback.onSuccess(true);
        }
    }
}

```

<hr />

App.java
```java
package com.samsung.multiscreen.msf20.helloworld;

import android.app.Application;

/**
 * @author plin
 *
 * The Hello World app.
 * 
 */
public class App extends Application {
    public static final String TAG = App.class.getName();

    private static App instance;
    private static Config config;

    private HelloWorldWebApplicationHelper msHelloWorld;
    public static App getInstance() {
        return instance;
    }

    public App() {
        instance = this;
    }

    @Override
    public void onCreate() {
        super.onCreate();

        config = Config.newInstance(this);

        msHelloWorld = HelloWorldWebApplicationHelper.getInstance(this);
    }

    public Config getConfig() {
        return config;
    }

    public HelloWorldWebApplicationHelper getHelloWorldWebApplication() {
        return msHelloWorld;
    }
}
```

<hr />

ActionBarConnenction.java
```java
package com.samsung.multiscreen.msf20.helloworld;

import java.util.HashMap;
import java.util.Map;

import android.graphics.drawable.StateListDrawable;
import android.view.View;
import android.view.View.OnClickListener;
import android.widget.ImageView;

/**
 * @author plin
 *
 * Manages the actionbar connect icon image view.
 * 
 */
public class ActionBarConnectIcon {

    public static final int LIGHT_THEME = 0;
    public static final int DARK_THEME = 1;
    
    private enum State {
        DISABLED_STATE(0),
        ENABLED_STATE(1),
        CONNECTING_STATE(2),
        CONNECTED_STATE(3);
        
        private final int state;
        State(int state) {
            this.state = state;
        }
        public int getValue() {
            return this.state;
        }

        private static final Map<Integer, State> intToTypeMap = new HashMap<Integer, State>();
        static {
            for (State type:State.values()) {
                intToTypeMap.put(type.getValue(), type);
            }
        }

        public static State fromInt(int i) {
            State type = intToTypeMap.get(i);
            return type;
        }
    };

    private static int[][] states = {
            new int[]{},
            new int[]{android.R.attr.state_enabled},
            new int[]{android.R.attr.state_enabled,android.R.attr.state_checkable},
            new int[]{android.R.attr.state_enabled,android.R.attr.state_checked}
    };
    
    private static final int[] themes = {
        R.drawable.mr_ic_media_route_holo_light,
        R.drawable.mr_ic_media_route_holo_dark
    };
    private View view;
    private ImageView imageView;
    private int theme = themes[0];
    private State state = State.DISABLED_STATE;
    private StateListDrawable slDrawable;
    
    public ActionBarConnectIcon(View view, int theme, 
            OnClickListener onClickListener) {
        this.view = view;
        if (view != null) {
            this.view.setOnClickListener(onClickListener);
            this.imageView = (ImageView)view.findViewById(R.id.connect_icon);
        }
        if ((theme == LIGHT_THEME) || 
                (theme == DARK_THEME)) {
            this.theme = themes[theme];
            this.slDrawable = (StateListDrawable)App.getInstance().getResources().getDrawable(this.theme);
        }
        
        setDisabled();
        
        if (this.imageView != null) {
            this.imageView.setVisibility(View.VISIBLE);
        }
    }
    
    public void setDisabled() {
        state = State.DISABLED_STATE;
        setState(states[state.getValue()]);
        makeClickable(state);
    }
    
    public void setEnabled() {
        state = State.ENABLED_STATE;
        setState(states[state.getValue()]);
        makeClickable(state);
    }
    
    public void setConnecting() {
        state = State.CONNECTING_STATE;
        setState(states[state.getValue()]);
        makeClickable(state);
    }
    
    public void setConnected() {
        state = State.CONNECTED_STATE;
        setState(states[state.getValue()]);
        makeClickable(state);
    }
    
    private void setState(int[] state) {
        if (imageView != null) { 
            slDrawable.setState(state);
            imageView.setImageDrawable(slDrawable.getCurrent());
        }
    }
    
    public void nextState() {
        state = State.fromInt((state.getValue() + 1)%states.length);
        setState(states[state.getValue()]);
    }
    
    private void makeClickable(State state) {
        switch (state) {
        case DISABLED_STATE:
            view.setClickable(false);
            view.setEnabled(false);
            break;
        case ENABLED_STATE:
            view.setClickable(true);
            view.setEnabled(true);
            break;
        case CONNECTING_STATE:
            view.setClickable(false);
            view.setEnabled(false);
            break;
        case CONNECTED_STATE:
            view.setClickable(true);
            view.setEnabled(true);
            break;
        default:
            break;
        }
    }

}

```

<hr />

ChannelListener.java
```java
package com.samsung.multiscreen.msf20.helloworld;

import com.samsung.multiscreen.Channel.OnClientConnectListener;
import com.samsung.multiscreen.Channel.OnClientDisconnectListener;
import com.samsung.multiscreen.Channel.OnConnectListener;
import com.samsung.multiscreen.Channel.OnDisconnectListener;
import com.samsung.multiscreen.Channel.OnErrorListener;
import com.samsung.multiscreen.Channel.OnReadyListener;
import com.samsung.multiscreen.Client;
import com.samsung.multiscreen.Error;

public abstract class ChannelListener implements OnConnectListener,
        OnDisconnectListener, OnClientConnectListener,
        OnClientDisconnectListener, OnErrorListener, OnReadyListener {

    @Override
    public void onError(Error error) {
    }

    @Override
    public void onClientDisconnect(Client client) {
    }

    @Override
    public void onClientConnect(Client client) {
    }

    @Override
    public void onDisconnect(Client client) {
    }

    @Override
    public void onConnect(Client client) {
    }

    @Override
    public void onReady() {
    }
}

```
