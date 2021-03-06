This is to fill the lacuna in the Yesod Book in the non-scaffolded MongoDB-department. The issue was raised  [over at StackOverflow](http://bit.ly/RzrwDI), as well as encountered by personal forays in Yesod.

**Update**: now works with yesod-platform 1.2.2.<br>
**Coming soon**: Showing what's already in the DB (The 'R' in 'CRUD').

``` haskell
{-# LANGUAGE OverloadedStrings, TypeFamilies, QuasiQuotes,
  TemplateHaskell, MultiParamTypeClasses, GADTs, FlexibleContexts #-}
import Yesod
import Yesod.Default.Config (DefaultEnv (..), withYamlEnvironment)
import Data.Text (Text, pack)
import Database.Persist (runPool, createPoolConfig, loadConfig,applyEnv)
import Database.Persist.MongoDB (MongoConf (..), Action, ConnectionPool (..), MongoBackend)
import Language.Haskell.TH.Syntax
import Control.Applicative ((<$>), (<*>), liftA2, Applicative)

data App = App ConnectionPool MongoConf

instance Yesod App 

instance RenderMessage App FormMessage where
    renderMessage _ _ = defaultFormMessage

instance YesodPersist App where
    type YesodPersistBackend App = Action
    runDB act = do
        App pool conf <- getYesod
        runPool conf act pool

share [mkPersist (mkPersistSettings (ConT ''MongoBackend)) { mpsGeneric = False }, mkMigrate "migrateAll"][persistLowerCase|
Questionnaire
  desc Text Maybe
  questions [Question]
  deriving Show Eq Read 
Question
  formulation Text
  deriving Show Eq Read 
|]

mkYesod "App" [parseRoutes|
/ RootR GET POST
|]

{-===========================================================================-}
{-                                  HANDLERS                                 -}
{-===========================================================================-}
appTitle :: Text
appTitle = "Non-scaffolded MongoDB-driven App skeleton"

getRootR :: Handler Html
getRootR = do
  (formWidget, formEnctype) <- generateFormPost choiceForm
  defaultLayout $ do
           setTitle $ toHtml appTitle
           toWidget [whamlet|
                     <h1>#{appTitle}
                     <form method=post action=@{RootR} enctype=#{formEnctype}>
                       ^{formWidget}
                       <input type=submit value="Send">
                     |]

postRootR :: Handler ()
postRootR = do
  ((result, formWidget), formEnctype) <- runFormPost choiceForm
  case result of
    FormSuccess rc -> do _ <- runDB $ insert rc
                         return ()
    _              -> return ()
  redirect RootR

{-===========================================================================-}
{-                             AN APPLICATIVE FORM                           -}
{-===========================================================================-}

choiceForm :: Html -> MForm Handler (FormResult Questionnaire, Widget)
choiceForm = renderDivs $ Questionnaire
               <$> aopt textField "Description of questionnaire: " Nothing
               <*> questionFields
             where
               questionFields = (questionAFormTextField "Question 1: ")
                                `applListAppend`
                                (questionAFormTextField "Question 2: ")
                                `applListAppend`
                                (questionAFormTextField "Question 3: ") 
               questionAFormTextField :: FieldSettings App
                                      -> AForm Handler [Question]
               questionAFormTextField fss = (:[]) <$>
                             (Question <$> areq textField fss Nothing)
               -- to append the list inside of the applicative
               -- context, i.e the `[Question]` - see type declaration
               -- above for `questionAFormTextField`.
               applListAppend = liftA2 (++)

{-===========================================================================-}
{-                                 MAIN                                      -}
{-===========================================================================-}


main = do
    dbconf <- withYamlEnvironment mongoConfFile Development
                Database.Persist.loadConfig >>=
              Database.Persist.applyEnv
    pool <- Database.Persist.createPoolConfig (dbconf)
    warp 3000 $ App pool dbconf
    where mongoConfFile = "./mongoDB-nonscaffold.yml" 
```
The configuration file is just taken from a scaffolded site and changed to fit our needs. This route seemed the easiest for me at the time (although, had `MongoAuth` been exported from `Database.Persist.MongoDB`, I would have a hand-made `MongoConf` above). It should be saved under whatever file `mongoConfFile` points to in the source above, and contain the following:
``` yaml
Default: &defaults
  user: <usr>
  password: <pwd>
  host: localhost
  database: yesod
  connections: 10

Development:
  <<: *defaults

Testing:
  database: yesod_test
  <<: *defaults

Staging:
  database: yesod_staging
  connections: 100
  <<: *defaults

Production:
  database: yesod_production
  connections: 100
  host: localhost
  <<: *defaults

```